# motor


This is a fork of
https://github.com/epics-modules/motor/

# Changes to upstream motor, the most important ones

## v7.0.3-ESS

###  Bug fixes

####  54c4286b: LOAD_POS: Ignore MRES when driver uses EGU
####  146f43ee: Improve handling of ramp-down after stop
####  146f43ee: Record recognizes motor stop while jogging


### Improvements

#### 19dc53c1: devMotorAsyn: Introduce Debug prints

## v7.0.2-ESS

###  Update to the latest motor/master, 2020-08-06, Includes upstream R7-2-1

###  Bug fixes

####  7546277f: JVEL while jogging, MF_DRIVER_USES_EGU is used
####  c92efe25: Late connect: Don't mis-use UDF, use neverPolled

### Improvements

####  RSTM field (Restore mode)
    The RSTM field is in upstram master,
    but today not part of an official release

####  b64af248: .SPAM field: Initialize it in motorRecord.dbd
    The motorRecord state machine is sometimes hard to follow.
    In order to make sure that we can follow all the transitions
    (especially when the driver reports "DONE") there is a new logging
    from inside the motorRecord itself.
    A simplified version of a move triggered by writing to the .VAL field
    of the PV "IOC:m1" looks like this:
    2020/06/26 13:29:15.370 [motorRecord.cc:1833 IOC:m1] doRetryOrDone dval=158.9 rdbd=0.1
    2020/06/26 13:29:15.370 [motorRecord.cc:1853 IOC:m1] mipSetBit 'Mo' old='' new='Mo'
    2020/06/26 13:29:15.375 [motorRecord.cc:1462 IOC:m1] msta.Bits.RA_DONE=0
    2020/06/26 13:29:15.487 [motorRecord.cc:1442 IOC:m1] msta.Bits.RA_MOVING=1
    2020/06/26 13:29:18.119 [motorRecord.cc:1442 IOC:m1] msta.Bits.RA_MOVING=0
    2020/06/26 13:29:18.119 [motorRecord.cc:1462 IOC:m1] msta.Bits.RA_DONE=1
    2020/06/26 13:29:18.119 [motorRecord.cc:1574 IOC:m1] motor has stopped drbv=158.970
    2020/06/26 13:29:18.119 [motorRecord.cc:1231 IOC:m1] maybeRetry: close enough; rdbd=0.1diff=-0.07 mip=0x20('Mo')
    2020/06/26 13:29:18.119 [motorRecord.cc:1234 IOC:m1] mipClrBit '...' old='Mo' new=''

    If you don't want these printouts, set the .SPAM field to produce less loggings

#### d7b19882: motorRecord.cc: Print the date and time in Debug()

## v6.9.7-ESS

### Updates from upstream, motor R7.0

####  3b3625b7c84 "Remove the config from controller code"
    When most (configuration) values can be read from the controller,
    this was possible to push those values into the motorRecord fields.
    (For example the default velocity was copied into VELO)
    
    Remove all this logic from the record and asyn device support.
    If needed, the driver can set up ai records which can be forwarded
    into the record fields with help of ao records.
    
    The only parameters that are left are the "read only softlimits".
    They are needed to restrict DHLM and DLLM inside the working range
    of the axis.

#### db759583 "motorRecord.cc: Correct SPDB handling when MRES == 1.0"

#### ae5e1fae "Add a SPAM field"
    Make the enable/disable of printouts more fine-grained,
    now we use a bit mask in the new introduced SPAM field:
    
     1 STOP record stops motor, motor reports stopped
     2 MIP changes,  may be retry, doRetryOrDone, delayReq/Ack
     3 Record init, UDF changes, values from controller
     4 LVIO, recalcLVIO
     5 postProcess
     6 do_work()
     7 special()
     8 Process begin/end

#### 36177f7b "motorRecord.cc: Add ACCS field"
    The problem:
    When a user changes the velocity by writing to the VELO field,
    but forgets to write to the ACCL field, the acceleration will
    change and may be too high.
    (Note: Many users are not aware of this side-effect).
    
    Solution:
    Introduce a field called ACCS, "Acceleration in seconds^2".
    
    Once the field is written, the ACCL field is updated and the
    acceleration is "locked".
    It is locked for changes of VELO.
    It can be be unlocked by writing to ACCL.
    
    If the acceleration is locked (or not) is stored in the field ACCU,
    which can be either "motorACCSused_Accl" or "motorACCSused_Accs".
    
    And in this sense ACCU is not a lock, but just keeps track which
    of the accelation fields ACCL or ACCS has been updated last, and
    should be used in the next movement.
    In any case an update to ACCS updated ACCL and the other way around.
    Thanks to Mark Rivers for this nice idea.

#### a01e46ef "Don't allow DHLM=DLLM=0 when read-only limits are set"

####  b912ecc7 "Move EGU"
    Make it possible to forward movements (absolute/relative) from the
    record into the model 3 driver:
    - Add a new device support function, moveEGU
    - Add it to the device suppport table for asyn motors, entry #9
    - Use writeGenericPointer in asynMotorController to receive
      the new command.
    - Add moveEGU() in asynMotorAxis, this function can be overloaded
      by a driver.
    - Add moveVeloEGU() in asynMotorAxis, this function can be overloaded
      by a driver.

## v6.9.6-ESS

### Updates from upstream, motor R6.11

Beside many bugfixes and improvements, here some high lights:

#### 8eacc2f0 "Adapt motorRecord.cc to base-3.16 with typed rset"

#### ee9fca1d "Fixes for problems that showed up in base 7.0. -
     It was using the value of mr->[link]->value.constantStr
     and mr->[link]->value.pv_link.pvname without checking the
     link types.   This was incorrect, and led to errors because
     constantStr no longer has a defined value if the link type
     is not CONSTANT.   Changed so that it only accesses these
     union fields if the link is the correct type.
     - It did not check if pv_link.pvname was NULL before calling
       CA functions, though now that we check the link type perhaps
       this cannot occur.
     - It did not lock the motor record before accessing the fields.
     - Added Debug calls to show the link types and values."

#### 95c0a4ca "Don't set "stop" field true if driver returns RA_PROBLEM true.
     It is the driver's responsiblity to stop a motor if the condition
     that results in the RA_PROBLEM bit being set doesn't result
     in the motor automatically stopping."


#### 7493d50b "If URIP=Yes and reading RDBL causes LINK error,
     do not start a new target position move."

#### 8eacc2f0 "Adapt motorRecord.cc to base-3.16 with typed rset"

#### 689a813a "Merge branch 'add_SPDB_v1'",
     Rename SDBD into SPDB

#### f3d3d2cb 'Add Adjust after homed'
    When homing against a limit switch, we find the motor position
    most often outside the valid soft limit range.
    Make it possible to move the motor inside the allowed range,
    by settting the "motorFlagsAdjAfterHomed_" parameter to 1.

#### 9c2c0c6f "STOP when hitting a limit switch is configurable"
    
    When a limit switch is hit when traveling into the commanded direction,
    the motorRecord sends a STOP.
    There are cases, when this is not desired:
    - The IOC is running while the motion controller is commissioned.
      The  motor is moved using an engineering tool,
      The motorRecord does not have a valid "commanded direction".
      The motion controller will stop the motor, when a limit switch is
      hit, and this shouldbe tested manually.
    - The motor is homed against a limit switch.
      The controller will stop the motor, and then move it slowly away from
      the switch.
      Once the switch releases, the current position is latched.
    
    Note:
    Different controllers and use cases may need different behaviour(s),
    so more commits and changes may come in the future.

#### Late connect
    Improve the situation when an IOC is started a long time before the
    controller "goes online".
    The basic idea is to keep the record in "UDF".
    Different commits like
    84d0d09b "devMotorAsyn: Keep record in longer in UDF if needed"
    
    When the communication with an asyn motor has not be established when the
    record is iniitialized, keep the record in UDF.
    (The record processing can not wait here)

#### Controller uses EGU
    Modern controller use engineering units, so that the IOC uses e.g.
    "mm" instead of "steps" when communicating positions.
    The motorRecord insists somewhat to use steps.
    As a result, both MRES must be set to some value, and then
    the controller must have a "scale factor", "stepsPerUnit" or the like.
    Solution:
      setIntegerParam(pC_->motorFlagsDriverUsesEGU_,1);
    Note that there is still a need to set either MRES or SPDB to a useful
    value, otherwise the motorRecord will refuse to move distances below
    1.0 mm.

#### Read-only soft limits from controller
    The model 1 and model 2 drivers can refuse to accept soft-limits.
    The soft limits in the motorRecord stay always inside the physical
    driving range of the axis.
    Example: The controller has a driving range 0..175 mm
    The user tries to set the DHLM field to 180mm.
    The device support in the driver is allowed to refuse the change,
    so that the DHLM field stays ar 175.
    Similar for DLLM, and the user coordinate-ish LLH and LLM
    This has never been implemented for model 3 drivers.
    If available, the "read only softlimits" can be written
    into the parameter library like this:
      pAxis->setDoubleParam(motorHighLimitRO_, limitHigh);
      pAxis->setDoubleParam(motorLowLimitRO_,  limitLow);
    And device support will forward them to the motorRecord.

#### Homing ignores limit switches
    The motorRecord assumes that there is a "home switch",
    which needs to be activated.
    The user must know, where the motor is, and decide if she wants
    to use HOMF or HOMR to initiate the homing.
    This does not work well if
    - (a) The limit switch itself is the "home switch"
    - (b) The homing sequence is like this
          "go to the low limit switch, activate it, stop, reverse the
          direction and search for the home switch.
    In case of (a) will the motorRecord send a stop to the controller,
    which may abort the homeing and leaves the axis in an unusable state.
    In case of (b) the same problem occurs.
    Solution:
      setIntegerParam(pC_->motorFlagsNoStopOnLS_, 1);
    There is another, sometime annoying, thing:
    When we have a controller, that can do a homing even when a limit
    switch is active, the motorRecord will ignore HOMF when HLS set.
    It silently ignores HOMR when LLS is active.
    So again, the user must choose between HOMR/HOMF.
    Solution:
      setIntegerParam(pC_->motorFlagsHomeOnLs_, 1);

#### Enhanced auto-power-on
    There are 2 improvements in the new "auto power on mode 2"
    The old auto power on had 0 (=off) and 1 (=on).
    The new mode "2" has two more features:
    - setting a power off timout < 0.0 will leave the axis
      powered on.
    - The fixed power on delay can be shortened, if the axis
      reports "power on achieved" earlier.
      A better documentation of this feature is needed.

#### All MRES related calculation are in motorDevSup.c
    The code in the motorRecord was full of DVAL/RVAL calculations,
    dividing by MRES was needed everywhere.
    This version uses dial coordinates in the motorRecord.cc
    and let motorDevSup.c convert them into "raw" values. 

#### Compiler warnings removed
    Removed all compiler warnings in the motor module.
    (Those that my compilers showed)

