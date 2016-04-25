# Writing games for Android in Haskell

## Sean Seefried & Christian Marie

---
# Before we start

* Happy if you get what you wanted out of this workshop and leave half way to go check out
  another
* These slides available at http://seanseefried.com/ylj16
* Start at https://github.com/sseefried/docker-game-build-env

---
# Part 1: Building on Android

---
# Basic architecture
* Build the game as a static library with entry point haskell_main (exposed via GHC FFI)
* Link all libraries (C and Haskell) into one big shared library. (ndk-build does this
  for you)
* Run a “small” Java program which uses JNI to load that shared library and call some C code.
* C code, in turn, calls into Haskell

???
- The first thing you do is build your game as a static library with entry point haskell_main
exposed via the Haskell FFI.
- You then link all the libraries, both C and Haskell, into one big shared library. This is
  done for you by an Android NDK (that’s Native Development Kit) tool called idk-build
- The Java program not really “small” over 1000 lines and handles initialisation of graphics,
  input and audio. Ripped straight from an SDL example project for Android.
---

## `SDLActivity.java`

```Java
public static native void nativeInit();

public static void unpackAssets() { ... }
/* sets global variable called externalAssetsDir */
```

---
## JNI implementation of `nativeInit`

```Java
void Java_org_libsdl_app_SDLActivity_nativeInit(
        JNIEnv* env, jclass cls) {
  ... /* initialisation here */
  jfieldID fid  = (*env)->GetStaticFieldID(env,
                               cls, “externalAssetsDir",
                               "Ljava/lang/String;");
  jstring assetDir = (*env)->GetStaticObjectField(env,
                                 cls, fid);
  const char *path = (*env)->GetStringUTFChars(env,
                                assetDir, NULL) ;
  char *argv[3];
  argv[0] = SDL_strdup("SDL_app");
  argv[1] = SDL_strdup(path);
  argv[2] = NULL;
  status = SDL_main(2, argv);
  ...
}
```
???
- This is how you declare a Java method using the JNI.
- The only important thing to see here is that we’re pulling out a Java String called
  `externalAssetsDir` (using some C function that access Java objects) and converting it into a
  C string and passing this to SDL_main.
- But where is SDL_main defined?  [NEXT]

---
## `main.c`


```C
#include <HsFFI.h>
#include "SDL_test_common.h"
extern void haskell_main(char *resourcePath);
int main(int argc, char *argv[]) {
  hs_init(&argc, &argv);
  haskell_main(argv[1]);
  hs_exit();
  return 0;
}
```
???
- Here. The inclusion of “SDL_test_common.h” does something strange. This header file further
  includes SDL_main.h which would you believe uses a macro to rename “main” to “SDL_main”
- This neat little trick allows me to pass the assets dir (which contains fonts, music and
  sound effects) to the Haskell main

---

## `AndroidMain.hs`

```Haskell
foreign export ccall haskell_main" main :: CString -> IO ()

-- CString is path to assets dir
main :: CString -> IO ()
main = ...
```
---

# Part 2: Game Design



---
# The main loop

```Haskell
mainLoop :: IORef BackendState
         -> (FSMState -> Event -> GameM FSMState) -- event handler
         -> IO ()
mainLoop besRef handleEvent = loop $ do
  runFrameUpdate       besRef
  runInputEventHandler besRef handleEvent
  runPhysicsEventHandler besRef handleEvent
  when (not profiling) $ delayBasedOnAverageFramerate besRef
  logFrameRate besRef
  where
    loop :: IO () -> IO ()
    loop io = io >> loop io
```

* In sense `handleEvent` _is_ the game. It's in `Game` module

???

---
## `runInputEventHandler`

```Haskell
runInputEventHandler :: IORef BackendState
                     -> (FSMState -> Event -> GameM FSMState) -> IO ()
runInputEventHandler besRef handleEvent = do
  mbEvent <- getEvents besRef
  case mbEvent of
    Quit        -> exitWith ExitSuccess
    Events []   -> return () -- do nothing
    Events evs  -> runUntilFSMStateChange evs
  where
    runUntilFSMStateChange :: [Event] -> IO ()
    ...
```
???

---
# Running events

```Haskell
runUntilFSMStateChange [] = return ()
runUntilFSMStateChange (ev:evs) = do
    bes <- readIORef besRef
    let fsmState = besFSMState bes
        gs = besGameState bes
    (fsmState', gs') <- runGameM (besGfxState bes) gs
                                 (handleEvent fsmState ev)
    writeIORef besRef $ bes { besGameState = gs'
                            , besFSMState  = fsmState' }
    if fsmState == fsmState' then runUntilFSMStateChange evs
                             else return ()
```
* Multiple events can be returned. Event could cause FSM state
  to change e.g. level to finish.
* When this happens _must_ flush the rest of the events


???

Multiple events can be returned. It's possible that one
of those events could cause a level to finish. When such a FSM state
change occurs we must flush the rest of the events.

---
# The game FSM

```Haskell
handleEvent :: FSMState -> Event -> GameM FSMState
handleEvent fsmState ev = do
  -- events that can occur in any FSM State
  case ev of
    Reset  -> resetGameState >> (return $ FSMLevel startLevelGerms)
    Pause  -> return $ FSMPaused fsmState
    _  -> (case fsmState of -- events that depend on current FSM State
             FSMLevel i                 -> fsmLevel i
             FSMPlayingLevel            -> fsmPlayingLevel
             FSMAntibioticUnlocked t ab -> fsmAntibioticUnlocked (t,ab)
             FSMLevelComplete      t    -> fsmLevelComplete t
             FSMGameOver           t    -> fsmGameOver t
             FSMPaused             s    -> fsmPaused s
           )
```

???
FSM = Finite State Machine

---
# Processing "Playing Level" state

```Haskell
fsmPlayingLevel :: GameM FSMState
fsmPlayingLevel = do
  gs <- get
  if M.size (gsGerms gs) > maxGerms
   then do
     t <- getTime
     addToSoundQueue GSLevelMusicStop
     return $ FSMGameOver t
   else do
    case ev of
      Tap p            -> playingLevelTap p
      Select p         -> playingLevelSelect p
      Unselect p       -> playingLevelUnselect p
      Drag p p'        -> playingLevelDrag p p'
      Physics duration -> do
        physics duration
        return fsmState
      _ -> return fsmState
```
---
# Events

* Abstracted away from low-level SDL events.
* Two layers
  - `Event` type in module `GameEvents`
  - module `Backend.Events` converts SDL events to game events

```Haskell
data Event = Tap R2       -- location at which tap occurred.
           | Select R2
           | Unselect R2
           | Drag R2 R2
           | Physics Time -- how much time the last frame took
           | Reset
           | Pause    -- home button pushed
           | Resume   -- back in the game
```

* Just what is a tap? How do you distinguish it from a drag?
* `Physics` is a "special" event handled by `runPhysicsEventHandler` in back-end.
???
---
# Events (continued)

* a "tap" is detected as
  - mouse/touch down occurs
  - less than `maxTapDuration = 200` milliseconds has elapsed
  - mouse/touch up occurs
* a "select" is detected as:
  - mouse/touch down occurs
  - greater than `maxTapDuration` milliseconds has elapsed
* An "unselect" occurs any time a mouse/touch up occurs
  after `maxTapDuration` milliseconds has elapsed since mouse/touch/down
* a "drag" happens anytime a mouse-motion occurs

---
# Graphics

<img src="germ.png">
* Germs design by Rauri Rochford
* Procedurally generated with Cairo in module `Graphics`
* Germ "arms" wiggle using sum of random periodic functions

```Haskell
-- From module Types.Basic
-- \t -> amp * sinU ((t + phase)/period)
--                  amp   phase period
type PeriodicFun = (Frac, Frac, Frac)
type MovingPoint = ((Frac, PeriodicFun),(Frac, PeriodicFun))
```
* See `movingPtToPt` in module `Graphics` for more detail

???
- Let me explain how this germ was drawn. For the outside imagine a polygon that looks like a
  star. We then use the midpoint of each arm and the tips of each star as control points and
  draw bezier curves. This smooths everything and the continuity guarantees of Bezier curves
  ensure that everything joins up smoothly.
- For the body of the germ the polygon was just random convext polygon that also got smoothed.


---
# Graphics

* Drawing a germ each frame then displaying it was far too slow
* Used OpenGL's (actually GL ES's) mip-mapping with trilinear filtering
* Wiggled the OpenGL polygon points instead of doing it in Cairo

```Haskell
germGfxRenderBody :: GermGfx -> Double -> Render ()
germGfxRenderBody gg r = do
   asGroup $ do
     scale r r
     translate 1 1
     withGradient (germGfxBodyGrad gg) 1 $ do
       blob . map (ptToCairoPt . movingPtToStaticPt) . germGfxBody $ gg
     -- scale to radius [r]
```

---
# GLM monad

* Graphics encapsulated in GLM monad
* GL ES uses GLSL shaders and cuts out much of OpenGL
* GLM monad hides lots of state such as:
  - compiled and linked GLSL shaders
  - FBOs and VBOs
  - font face
  - GL co-ordinate system bounds
* See module `Graphics.GLM`
* Don't mess with this stuff for this workshop. Too error prone!

```Haskell
data GLM a = GLM { unGLM :: GfxState -> IO a }

data GfxState = GfxState { gfxBlurGLSL    :: GLSLProgram BlurGLSL
                         , gfxWorldGLSL   :: GLSLProgram WorldGLSL
                         , gfxFontFace    :: FontFace
                         , gfxMainFBO     :: FBO -- on iOS this is not 0!
                         , gfxScreenFBId  :: FrameBufferId
                         , gfxTexturePool :: IORef [TextureId]
                         }
```

???
- Mipmapping is a pretty wonderful techcrenique. You take a texture, and the draw it at various
powers of two resolution. i.e.. 512x512, 256x256, 128x128 etc, all the way down to 1x1.
You can then use trilinear filtering to smoothly scale the texture to any size in between.
The result is not as good as drawing each frame from scratch, but it is good enough.

---
# Co-ordinate systems

* 4 co-ordinate systems in the game! See module `Types.Contants` for
  lengthy explanation in comments
* They are: screen, normalized, canvas and world
* module `Game` works exclusively in _world co-ordinates_
* The _world_ is split into a _side bar_ and a _game field_

```Haskell
--    The game field will always have bounds
--      (fieldLeft,       fieldRight,    fieldBottom,      fieldTop)
--    = (-fieldWidth/2, fieldWidth/2, -fieldHeight/2, fieldHeight/2)
fieldWidth  = 100
fieldHeight = 100
```
???
- screen: SDL Mouse input comes in this co-ordinate system.
- normalized: SDL touch input
- canvas: Cairo graphics
- world: co-ordinates that `Game` module deals with

---

# Part 3: Building on VM

---
# Why Docker?











---
# Running the Docker container

```
$ cd ~
$ docker images

REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
android-haskell                       latest              f1642cb749fa        40 hours ago        6.083 GB

$ ./run-env.sh android-haskell
```

Also can do

    $ ./run-env.sh f1642cb749fa

Can tag images with

    $ docker tag <image id> <tag>

e.g.

    $ docker tag f1642cb749fa android-haskell


---
# Building packages

## Local testing (on host machine)

```
$ cd ~/hip-code/open-epidemic-game
$ stack build
$ stack exec Epidemic
```

## Building for Android (in Docker container)

1. `$ cd ~/hip-code/android-build-game-apk`
2. Set up `config.json`
3. `$ ./build-game-apk.sh`


---
# Exercises

1. Make the germs "shirk away" from a death nearby. How quickly they move should
   be proportional to how close the death/tap was to them

2. Instead of having germs just divide in two, make them divide into a random
   number of germs

3. Add gravity into the game. Create a beaker using a few `AddHipStaticPoly`s from the `HipM`
   monad
---
