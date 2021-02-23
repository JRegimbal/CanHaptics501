## Lab 4: Controlling Actuation using PID

### Overview

### Tuning

First it is necessary to tune the PID controller in the order of P, D, I.
Note that I made some changes to the tuning control of the skeleton project
adjusting the increments of some values and also updating the on-screen values
when the keyboard is used to adjust them. These changes can be seen in the `keyPressed()` method.

#### P

The P parameter controls the proportional response of the controller relative to distance from the target.
I noticed a trend where at low values (~0.01), the end effector would move slightly closer to the destination, but would still be a centimeter or so from the target.
Increasing the value, however, to 0.002 or 0.003 causes the end effector to overshoot the target in some cases.
If it is too high, the end effector moves too quickly, the position is not tracked correctly, and the Haply gets stuck with the end effector pushing towards the edge.

I decided to set the value of P to 0.01 to avoid cases where it would overshoot and the position tracked by the software would not agree with the actual location of the end effector. This might change if the loop frequency increases and these quicker movements are tracked.
However, at this current value, the end effector moves closer to the target. It does not overshoot and stops
before reaching it as friction overpowers the force signalled by the controller.

<video width="100%" controls>
  <source src="../assets/lab4/P-0.01.webm" type="video/webm">
</video>

#### D

I began increasing D until a value of 1.20. A smoothing value of 0.97 was used based on the tip.
The Haply does get closer to the actual desired value (the end effector symbol overlaps with the target in many cases as shown below), but some oscillatory behavior does appear.
Usually this behavior dies off quickly, but in some cases the end effector moves back and forth or
ends up not properly tracked by the software and needs to be reset due to sudden movements.

<video width="100%" controls>
  <source src="../assets/lab4/PD-1.20.webm" type="video/webm">
</video>

Notably a steady state error does persist in many cases, although generally the end effector does get closer to the target much more quickly.

#### I

The integrator part was more difficult to tune as by default the integrator will run until manually reset.
This creates problems when a steady state error exists and I is set to 0. To make testing a bit easier,
I set the running sum to reset when a new random point is used.

From trial and error, I found good results with a parameter of I=0.03. Increasing above 0.03 results in
overshooting the target.
The PD part of the controller
gets close to the desired target and then any steady state error is compensated with the integrator. This
can be seen in the video below.

<video width="100%" controls>
  <source src="../assets/lab4/PID-0.03.webm" type="video/webm">
</video>

At this point, I'm not seeing the kinds of oscillations previously reported for the PD controller, however they were not extremely common in the first place and may still be possible.

#### Updating PID and Looptime

Before getting into tracking a moving target, I'd like the controller to be as good as possible and reach the
desired position faster if possible.
Since it seems many problems come from the end effector overshooting and getting into a bad state, I think
increasing the update frequency might prove useful.
However,the keypresses do not seem to actually adjust the looptime parameter correctly and the next lowest
value on the slider, 250 ms, is too quick a loop for Processing on my computer.

Next was adjusting the PID values to try and result in reaching the target more quickly.
Now that the integrator and differentiator parts are present, I increased the proportional value to
0.035. The integrator was kept at 0.03, but the differentiator had its value increased to 1.40 and the smoothness
decreased to 0.80 from 0.97 since I noticed hysteresis happened fairly regularly with that higher level of smoothness.

This results in the behavior below.

<video width="100%" controls>
  <source src="../assets/lab4/PID-v2.webm" type="video/webm">
</video>

Increasing the loop time (decreasing the loop frequency) results in heavy oscillatory behavior as the end
effector overshoots, updates, then turns around to correct. This behavior is shown in the video below.
At the end, the position tracking ceases to be accurate and the end effector is stuck in the real world.

<video width="100%" controls>
  <source src="../assets/lab4/PID-overshoot.webm" type="video/webm">
</video>

Attempting to decrease the loop time to 400 ms does result in some smoother motion, but on my computer
at least the tracking will break once the loop underruns which does not take very long. Then the program
needs to be restarted.

### Path Tracking

Path tracking was implemented where a circle is drawn around the randomly chosen point.
The target point moves around the circle at 30 rpm and the circular path has a radius of 0.3 in the
physical coordinate system. The target position is updated as part of the drawing loop.

The actual tuning parameters needed to be updated. I noticed that the high differentiator value
resulted in lots of vibration coming from the motor and the proportional value was too low to really follow
the moving point once it got close enough.
The new values used are P=0.06, I = 0.03 (no change), D=0.60, and smoothness is 0.80. Loop time was kept at 500 since that is the fastest that can reliably be run.
With these parameters, the end effector roughly follows the target along a circular path.

<video width="100%" controls>
  <source src="../assets/lab4/PID-circle.webm" type="video/webm">
</video>

The proportional value may be too high considering that some oscillations still occur, however it seems
that there is a trade-off between getting close to the target when the center of the circle moves and
then closely following the circular path.

### Conclusion
