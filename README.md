# Transform Quick Sync

or TQS for short is an improvement of the [15 Bits Position Rotation Networking](https://github.com/VRLabs/15-Bits-Position-Rotation-Networking) prefab by [VRLabs](https://github.com/VRLabs).
compared to VRLabs implementation this prefab uses less bits, fewer contacts, less physbone's and has some speed increases.
aside from these improvements the system also has another benefit. it allows you to sync up to 255 different transforms.
while this might take a bit to set up it allows for some pretty efficient configurations.
note that only 1 transform at a time can be synced, so syncing all transform might take a bit longer.
that's why it's smart to only sync transforms that are active. 

do note that this prefab is aimed at the more knowledgeable creators.
it can take some skill to both set this up and control this system as it does not have a "out of the box" ready setup (this might be added later if i can get it that far).

comparisons:

| compared         | 15Bit | TQS  |
|------------------|-------|------|
| Contacts         | 5     | 2    |
| Physbone's       | 2     | 1    |
| Collider         | 1     | 1    |
| Synced bits      | 15    | 12   |
| Animator layers  | 1     | 2    |
| sync speed (sec) | 6-13  | 4-7* |
| Constraints      | 19    | 5    |
| Objects          | 18    | 12   |

*because syncing is halted when updating the transform the first syncing cycle takes about 1 to 3 seconds longer. this is not included in the table.


## how it works
the animator controller is divided in 3 sections: fetching, transmitting, receiving.

### fetching
when the `get_pos()` parameter is set the animator will start fetching the transform and then store the individual values in multiple parameters.
after all values are fetched some checks will be performed to ensure the data is correct. it will first check for overflows and correct those.
then it checks if the fetched positions is within a small area af the actual position. if not it will try again.

### transmitting
transmitting happens automatically while `get_pos()` is false. in this state the animator will keep sending all 13 parameters.
each cycle typically takes less than 4 seconds to complete.

### receiving
this is the state that remote instances of your avatar will always be in. the previous 2 states were exclusive to your local instance.
while receiving the animator is always updating its values. this is done via the sync table found below.


## setup
setting up the system is the hard part, but is still quite easy.

first you will need to merge the TQS controller with your own FX controller.

after that add the `transform quick sync` prefab to the root of your world constraint.

now the difficult part.

### setting up tracking constraints.
here we will set up the constraints that come with the TQS prefab. 
- step 1: in your hiearchy go to the `transfrom quick sync` gameObject and expand it.
- step 2: expand `position tracker` and select `pos`.
- step 3: add all the transforms you want to sync to the Sources list of the position constraint and set their weight to 0.
- step 4: expand `rotation tracker` and select `full rotation`.
- step 5: repeat step 3 for the rotation constraint. **ensure the order is the same as in the position constraint**.

### modifying your existing constraints.
now we need to modify your constraints, so they can copy the synced transformation. it might be smart to make a backup or duplicate your avatar in case something go's wrong.
- step 1: add an extra entry to your sources list.
- step 2: put the "client transform" object in this new entry and set its weight to 0. ensure this is the last entry as to not break any existing animations.
- step 3: repeat steps 1 and 2 for all objects you want to sync. the order you do this in does not matter.

### preparing animations.
each synced transform needs 2 animations. 1 that is used on the local side and 1 for the client side. here we will be creating the animations and later on they will be edited.
- step 1: for every transform you want to use duplicate the `host target template` animation in the target's folder(you can rename it if you want. ex: `host target 1`, `host target 2`, etc.).
- step 2: do the same for the `client target template`.
- step 3: navigate to the `TQS blendtrees` layer in **your** animator.
- step 4: enter the state `local` and select the `set target` tree.
- step 5: click on the `+` and select `add motion field`. do this the same amount of times as animations you had created.
- step 6: now drag all the animations for the host (`host target 1`, `host target 2`, etc.) **again making sure your order is the same**.
- step 7: return to the `TQS blendtrees` layer and enter the `remote` state.
- step 8: repeat step 5 and 6 for the `set pos` tree and with the client animations.

### editing the animations.
this is the second to last step, and it involves editing the animations we just created and added to the animator.
- step 1: start recording on the host animation you want to edit.
- step 2: go to the pos constraint that is referenced during the "setting up tracking constraints" part of the setup.
- step 3: animate the weight of the Source which has the same index as its index in the blendtree to 1. (eg: if this is the 3rd animation in the `set target` blendtree you should animate the 3rd source of the position constraint.)
- step 4: repeat step 2 and 3 for the rotation constraint.
- step 4: end recording this animation and now start recording the client animation instead.
- step 5: select the constraint that should be synced by the current index.
- step 6: animate the `enable constraint` to on, animate the constraint weight to 1, animate all the Source weights to 0 except for the last index (the one from our client transform) and set it to 1 instead.
- step 7: repeat for all animations you have created.

### adding defaults.
as the last step we need to at some defaults. this should be short.
- step 1: start animating the `host target default` animation.
- step 2: animate all Source weight of the position and rotation constraints to 0 (these are again the same constraints that a referenced in "setting up tracking constraints").

## how to use TQS.
now that the setup is complete you can start using the system. using the system is pretty simpel. there are 2 parameters you can use to control the system.
these are `float TQS/target` and `bool TQS/get_pos()`.

`TQS/target` is the index of the transform you currently want to sync. this ranges from 0 to 255.
index 0 will not sync any transform. this means you can disable the system by setting the index to 0.
for any other value up to 255 it will use the transform that you defined during the setup.

`TQS/get_pos()` will update the transform and the index. you need to set this bool if you changed the index or your current transform has changed.
if you do not call it the system will continue to sync the last stored transform and index. you do **not** have to set this back to false.
once the system starts the fetching phase it will automatically reset this bool.


# sync table.

this table shows what the system is doing for each state of the steps.
you can also reference this table if you plan on making your own syncing method.


| step | bit0 | bit1 | bit2 | bit3 | action          |
|------|------|------|------|------|-----------------|
| 0    | OFF  | OFF  | OFF  | OFF  | idle            |
| 1    | ON   | OFF  | OFF  | OFF  | sync X rough    |
| 2    | OFF  | ON   | OFF  | OFF  | sync Y rough    |
| 3    | ON   | ON   | OFF  | OFF  | sync Z rough    |
| 4    | OFF  | OFF  | ON   | OFF  | sync X medium   |
| 5    | ON   | OFF  | ON   | OFF  | sync Y medium   |
| 6    | OFF  | ON   | ON   | OFF  | sync Z medium   |
| 7    | ON   | ON   | ON   | OFF  | sync X fine     |
| 8    | OFF  | OFF  | OFF  | ON   | sync Y fine     |
| 9    | ON   | OFF  | OFF  | ON   | sync Z fine     |
| 10   | OFF  | ON   | OFF  | ON   | sync Yaw        |
| 11   | ON   | ON   | OFF  | ON   | sync Pitch      |
| 12   | OFF  | OFF  | ON   | ON   | sync Roll       |
| 13   | ON   | OFF  | ON   | ON   | apply transform |
| 14   | OFF  | ON   | ON   | ON   | unused          |
| 15   | ON   | ON   | ON   | ON   | unused          |
