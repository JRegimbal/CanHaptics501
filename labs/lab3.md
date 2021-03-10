## Lab 3: Communicate Something with Haply

Thanks to Hannah E. and Rubia G. for testing!
Source code is available at as a [zip file](../assets/lab3/sketch_lab3_regimbal.zip).

### Pre-Implementation Planning

Prior to starting anything using Haply, I first started to come up with some ideas for what to express.
I understood the instructions as meaning that a high-level adjective should be conveyed rather than
doing something straightforward (e.g., having the end effector swirl around for "vortex").

![Image of first planning page](../assets/lab3/DOC-20210210-140018.jpg){: width="50%" style="margin: 0 auto; display: block;"}
*Brief planning descriptions of the effects. These are elaborated on below.*{: style="display: block; text-align: center;"}

I ultimately ended up coming up with three concepts to convey. Taking from the image above:

* **annoying/frantic**: A spring force in varying directions changing rapidly. Dynamic but random.
* **surprising/charming**: Changes but predictable after. Gives time to explore and reflect.
* **calm/content**: Constant/consistent with smooth transitions. Something like rounded edges, viscosity moving
from a center point.

Working from these starting points, I made some pen and paper sketches of what the interactions might look like.

![Image of pen and paper sketches and elaboration.](../assets/lab3/DOC-20210210-140058.jpg){: width="50%" style="margin: 0 auto; display: block;"}
*Initial rough descriptions and sketches for the ideas.*{: style="display: block; text-align: center; width: 75%; margin: auto;"}

The first basic sketch is for **calm/soothing**. This is described as being a static, viscous region.
The second is **annoying** which has a grid on the workspace of the Haply. A point on this grid is
randomly selected and a spring force is rendered from that point to the end effector. To make it especially
annoying, I decided the point should be updated every 500 ms.
The last is the surprising mode, which again uses a grid and spring forces. However rather than having
one point of the grid chosen randomly, all of them exert spring forces if the end effector is positioned
within a certain distance of it. The idea here is that the user would then try to explore the space and find the
different regions.

### Implemenation

#### Structure

The generic structure for a Haply program in Processing is followed in that a set of variables and constants are defined,
`setup` and `draw` functions are written, and a thread is spawned that handles physics simulation and the device.
For this app, Fisica was *not* used.

After some basic testing of the viscosity idea, I found that viscosity itself wasn't very compelling. Either it
was too strong or not really perceptible over the friction in the Haply itself. So, I tried setting the damping
factor to a small negative value to try and "help" along the end effector slightly when the user tries to move it.
My thinking is that by slightly reducing the apparent friction, the calm, smooth feeling would be conveyed.

A mode-based approach was taken to implement the three concepts. This was defined as
`enum Mode { CAL, ONE, TWO, THREE };` where `CAL` is calibration mode
(where a visual of the pantograph is shown and updated) and the other modes refer to the
numbered mode. In this case, mode one is calm, mode 2 is annoying, and mode 3 is surprising.

Values were set for the simulation building off of the "Hello Ball" example. Note that when different
adjectives the modes were supposed to convey were referred to in the code, they were hidden with [ROT13](https://en.wikipedia.org/wiki/ROT13)
in case anyone testing decided to look through. The parameter setup is as follows:

```java
/** Generic Parameters */
float rEE = 0.006;
float rEEContact = 0.006;
float dt = 1 / 1000.0;
/** Mode 1 Parameters (fbbguvat) */
float bFluid = -2.5;
/** Mode 2 Parameters (naablvat) */
float tLastChange = Float.NEGATIVE_INFINITY;
float tDuration = 500; // ms
PVector[] locations2 = {
  new PVector(-0.07, 0.03), new PVector(0.07, 0.03),
  new PVector(-0.07, 0.07), new PVector(0.07, 0.07),
  new PVector(-0.07, 0.11), new PVector(0.07, 0.11)
};
float k2 = 220; // N/m
int index2;
/** Mode 3 Parameters (fhecevfvat) */
PVector[] locations3 = {
  new PVector(-0.07, 0.03), new PVector(0, 0.03), new PVector(0.07, 0.03),
  new PVector(-0.07, 0.07), new PVector(0, 0.07), new PVector(0.07, 0.07),
  new PVector(-0.07, 0.11), new PVector(0, 0.11), new PVector(0.07, 0.11),
};
float k3 = 220;  // N/m
float xThreshold = 0.025;   // m

/** State info */
PVector angles = new PVector(0, 0);
PVector torques = new PVector(0, 0);
PVector posEE = new PVector(0, 0);
PVector posEELast = new PVector(0, 0);
PVector velEE = new PVector(0, 0);
PVector fEE = new PVector(0, 0);

PVector deviceOrigin = new PVector(0, 0);
```

The physics rendering was implemented in the following run function:
```java
public void run() {
  renderingForce = true;
  if (haplyBoard.data_available()) {
    widget.device_read_data();
    angles.set(widget.get_device_angles());
    posEE.set(widget.get_device_position(angles.array()));
    posEE.set(device_to_graphics(posEE));
    velEE.set((posEE.copy().sub(posEELast)).div(dt));
    posEELast.set(posEE);

    // TODO physics
    PVector force = new PVector(0, 0);
    if (mode == Mode.ONE) {
      force.set((velEE.copy()).mult(bFluid));
    }
    else if (mode == Mode.TWO) {
      // Check if should update random spring loc
      if (millis() - tLastChange > tDuration) {
        index2 = int(random(0, locations2.length - 1));
        tLastChange = millis();
      }
      PVector xDiff = (posEE.copy()).sub(locations2[index2]);
      force.set(xDiff.mult(-k2));
    }
    else if (mode == Mode.THREE) {
      for (PVector loc : locations3) {
        // Check if within threshold
        PVector xDiff = (posEE.copy()).sub(loc);
        if (xDiff.mag() < xThreshold) {
          // Add force
          force.add(xDiff.mult(-k3));
        }
      }
    }

    fEE.set(graphics_to_device(force));
  }

  torques.set(widget.set_device_torques(fEE.array()));
  widget.device_write_torques();
  renderingForce = false;
}
```

#### Debugging

One extremely obvious issue is that if a force is being exerted when the program ends, the Haply will
keep rendering that force on the device. This comes up a lot in modes 2 and 3.
To solve this, another variable was defined `ScheduledFuture<?> handle;` that then gets the handle
for the scheduled haptics thread.
```java
handle = scheduler.scheduleAtFixedRate(st, 1, 1, MILLISECONDS);
```
Then the function run on exit in Processing was overridden to ensure
the last message sent to the Haply before closing was to set torques to 0.
```java
void exit() {
    handle.cancel(true);
    scheduler.shutdown(true);
    widget.set_device_torques(new float[]{0, 0});
    widget.device_write_torques();
    super.exit();
}
```

### Testing

A brief survey was included in the instructions in the [README](../assets/lab3/README.txt)
packaged with the sketch and dependencies.
Essentially, it asked the tester to list three adjectives for each of the modes in the order they are most
applicable. Testers were also asked to specify a separate list if they believed that the modes were trying
to convey something different, but failing to.

The feedback was as follows:

| Tester # | Mode 1 (Soothing) | Mode 2 (Annoying) | Mode 3 (Surprising) |
| ---      | ---               | ---               | ---                 |
| 1        | subtle, smooth, small | chaos, wild, random | checkered, jumpy, magnetic |
| 2        | viscous, grainy, free | violent, wild, random | restrained, checkered?, "holey"? |

### Discussion Reflection on Attempt

This attempt yielded mixed results. The words given for the "annoying" mode aligned fairly well with the concept.
Both "wild" and "random" were used to describe the sensation and the descriptions tended towards negative valence high arousal, which I believe aligns with "annoying".
The results of the "soothing" mode were in alignment to a lesser extent. No words were matched across the testers unlike
for "annoying". However the words listed seem to convey low arousal which is the goal. In terms of valence, they tend to come across as neutral compared to the goal of "soothing".
Finally for "surprising", the aspect of surprise seems to have been lost. While aspects of the experience itself
were given (e.g., jumpy) it doesn't seem like surprise was invoked in either person, or if it was it wasn't
considered notable.

To try and provide a more general measure of success, the valence and arousal values for each word from
[Warriner et al.](https://doi.org/10.3758/s13428-012-0314-x) were compared between the source word for each mode and the words given by the participants.
Note that "viscous" wasn't in the dataset and was replaced with "gooey".
A summary of each value is given below. Valence and arousal are given on a 9-point scale.

| Mode Word | Mode Valence | Mode Arousal | Avg. Resp. V | Avg. Resp. A | Error V | Error A | &sigma; V | &sigma; A |
| ---       | ---          | ---          | ---             | ---             | ---     | ---     | ---       | ---       |
| Annoying  | 2.49         | 5.04         | 4.01            | 5.52            | 1.52    | 0.48    | 1.73      | 1.07      |
| Soothing  | 7.05         | 1.91         | 6.20            | 3.54            | -0.86   | 1.63    | 1.07      | 1.35      |
| Surprising| 6.75         | 5.65         | 5.02            | 3.97            | -1.74   | -1.69   | 0.63      | 0.92      |

These suggest the same trends observed above where the annoying mode and soothing mode were somewhat close to
the desired word while the surprising mode was more off. Also interesting to note is that while the errors for annoying
and soothing are within a standard deviation of the desired values for valuence and arousal, this is not the case for surprising.
In fact, the words given for surprising were more in agreement in valence and affect than for the others.

Ultimately, I noticed that conveying the subtleties of a concept solely using force feedback (e.g., with "suprising") was very difficult.
Even in cases where the general valence and affect were conveyed successfully, the finer points of it were more difficult to convey where there
isn't a literal point. It's possible that asking another question could have lead to the desired responses, however this suggests that these
aspects are not readily perceived by the participants and are difficult to design into a purely haptic experience.

While more mature haptic rendering techniques likely would have helped (the bulk of the work on this lab was done before the initial due date), I
think the main point I am taking away is how important context is to how a stimulus or set of stimuli are interpreted. Surprise depends
on having expectations and having them broken in some way. My motivation for using "annoying" as something to communicate was my cat yelling
while I tried to work, which itself is only annoying when I'm busy. At other times I appreciate how talkative she can be.

I suspect that more common haptic experiences would make this task easier --- typical use would establish some sort of base context --- but
I also think using different modalities would accomplish this. A scenario in everyday life is typically multisensory by nature, and among ones that
only leverage one sense there are relatively few that are solely haptic. Even adding in information in another sense would likely be enough
to add the context necessary for more complicated concepts to be conveyed.

A richer form of haptic feedback would also likely aid in this. [*Hand Movements*](https://doi.org/10.1016/0010-0285(87)90008-9) identified eight different exploratory procedures used to
identify an object. Most of these are not possible in 2 DoF force feedback through a pantograph, and those that are possible are limited in the
information that can be conveyed. A kind of haptic device that allows for more of these EPs would increase the number of possible interactions
and perhaps allow for more nuanced scenarios. I don't know how this would compare to adding another non-haptic modality, but I think it would
be worthwhile to explore it.

### Conclusion

Interactions to convey three different concepts were sketched out and implemented using Processing and hAPI.
The interactions were haptic-only and visuals were only used for debugging purposes.
Two participants were asked to try them out with no knowledge of the word they were intended to convey
and each wrote three words down they felt best described the experience. These words were compared to the
motivating words both based on subjective closeness and by valence-arousal values in a published dataset.
Two of thre three words were fairly close to the intended affect.

While getting somewhat close to an affect using a GFF-only experience is possible, conveying more nuanced
concepts seems to rely heavily on the perceived context of the experience. While this can probably be done
through richer haptic rendering, leveraging multimodal interaction is likely easier in most scenarios.
As an anectode, someone tried out the "Hello Ball" example on my Haply and noted (without prompting) that
it didn't feel much like anything just using the haptics alone, but when looking at the end effector and ball
on the screen the intended interaction was obvious and easy to understand. To me it seems that while haptic
interactions can be very powerful when designed well, expectations and content from other modalities are
also important when it comes to how a design is interpreted.
