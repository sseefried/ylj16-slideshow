# Writing games for Android in Haskell

## Sean Seefried & Christian Marie

---
# Preparation

* Copy the VM OVA off the USB stick and pass on to next person
* Double click the OVA and import the VM user/password is `android`/`android`
* These slides available at http://seanseefried.com/ylj16
* Once you've started the VM open README.md on desktop and have a read. You should aim to:
  - run Docker container
  - Build game APK in Docker container
  - Get VM to recognise your android device and enable "Developer options" menu
  - Deploy APK with `adb install`

---

# Demo of the games

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
???
- nativeInit needs to be implemented in C which I'll show in the next slide.



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


# Building an environment

## Building GHC cross compiler to ARMv7

* Full credit to neurocyte (CJ van den Berg)
* Builds iconv, ncurses and libgmp
* Applies patches
* Builds

???
- I can’t thank CJ van den Berg enough for this work. I can safely say that if this work had not
been done I may well have given up in despair at this point. At the very least it would have
slowed me down a lot.
- Just building common C libraries on Android can be difficult as the NDK only comes with
   “cut down” versions of many libraries. For instance glibc does not exist. Google replaced that
   with a cut down version called bionic
- The wrapper scripts

---

# Building an environment

Pure Haskell libraries were fine
It was the Haskell C binding libraries that were tricky

Haskell dependencies:

* SDL2: Window management and event handling
* SDL2-mixer: for sound
* Cairo: For procedurally generated graphics for the germs
* OpenGLRaw: to distort germs and perform other rendering effects
* HipMunk. A binding to Chipmunk for physics. Couldn't find a binding for Box2D (and didn't want to write one)

C dependencies:

* Taking account of further deps the complete list of C libraries was:
  cairo, freetype2, SDL2, SDL2_mixer, libogg, libpng, libvorbis, libvorbisfile, pixman, GLESv2
* These did not always build “out of the box” on Android and required patching

---

# Repeatable builds

* We know the complete build will take hours.
* Later stages depend heavily on earlier stages
* It’s brittle. One small change to any part could make it fail. This makes testing difficult!
* Losing a day because your 2 hour build script failed because of something
  stupid 4+ times is liable to send you into a rage and/or insanity!

???

The problem of building and developing a long build script

The first major problem is that we know it’s going to take hours to run. Later stages of the build
depend heavily on the earlier stages. Now here’s the problem. Even if you’ve correctly realised that
you should break your build script into smaller build scripts that you run in sequence, at some
point you’re going to run into a library build that doesn’t quite work the first time. So you hack a
little, patch it up, run make clean, and try again. But how truly know that the full state of the
filesystem is in exactly the state it was before you first ran the failed build?  You don’t! And so
this makes testing difficult because you have to run the damn script, from scratch, to be truly
sure.

---


# A novel use of Docker

_What if you could check-point the file system between phases of the build!?_

* Docker uses a union file system
* Docker has a build cache
* Dockerfile`s have ADD and RUN commands.
* ADD copies files from host to image
* Trick is: ADD a “scriptlet” just before you RUN it. Excellent for maintainability

???

What I’m about to show you relies on a feature of Docker that was introduced for reasons of
efficiency, but which is perfect for developing a long running build scripts.

Docker uses a union file system which is a essentially a file system in which files are layered on
top of each other. If a file in a higher layer is at exactly the same path as a file in a lower
layer then it shadows (or effectively overwrites it). In this way you can build up a file system as
a set of diffs in very much the same way as your favourite revision control system builds up text
files as a collection of diffs.

Now, those of us who have used Docker know that we build images using Dockerfiles. The fantastic
thing about Docker is that it maintains a build cache. This build cache effectively corresponds to
the layers of the union file system. Each “command” of the Dockerfile can be uniquely identified. If
you change, say, the fifth command then Docker will effectively only build onwards from that
command.

The ADD command is fantastic because it adds files from the host to the Docker image, effectively
introducing the files “just in time” to be compiled.  There is also a RUN command to run arbitrary
commands. The trick is to ADD libraries just before you RUN the commands to build them. This way you
are absolutely sure that the file system is a guaranteed state.

It also helps for maintainability’s sake. If you want to upgrade the library the Dockerfile will
only have to build onwards from the point where you introduce the new library files. If you’d added
the library files earlier in the Dockerfile you have no choice but to build from scratch! ARGH, that
takes forever!

---

# Docker's union file system

<img src="docker.png">

---

# Dockerfile example

```
$ docker build .
… stuff happened here…
Step 82 : ADD scripts/build-Hipmunk.sh $BASE/
 ---> Using cache
 ---> 2e3dd7c9429c
Step 83 : RUN ./build-Hipmunk.sh
 ---> Using cache
 ---> fec2714e936d
Step 84 : ADD scripts/clone-OpenGLRaw.sh $BASE/
 ---> Using cache
 ---> eed2b8d64ff6
Step 85 : RUN ./clone-OpenGLRaw.sh
 ---> Using cache
 ---> 016bf318c97e
```

The repo `https://github.com/sseefried/docker-game-build-env` might be a bigger
contribution to the community than the game itself.


# Part 2: Space Invaders & Functional Reactive Programming

---

[Add slides here]


---
# Exercises for Space Invaders


1. Make the game playable with a mouse. This will make it playable using a touch screen
2. Make the space invaders move right,down,left,down,right,down etc
3. Add bombs that can destroy the cannon and must be dodged
4. Make a mother ship come past every so often
5 (optional) Clean up the collision detection to be pixel perfect

---
# Part 3: Epidemic

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

- In order not to consume too many resources the `delayBasedOnAverageFramerate`
  function does exactly what you might think and just sleep for a certain amount of time
  but it does so on a sliding window basis


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

# Monads: HipM

See module `HipM`

```Haskell
data HipScript next =
    HipStep          !Double next
  | AddHipCirc       !Double !R2 (HipCirc -> next)
  | GetHipCircPos    !HipCirc (R2 -> next)
  | GetHipCircVel    !HipCirc (R2 -> next)
  | SetHipCircPos    !HipCirc R2 next
  | SetHipCircVel    !HipCirc R2 next
  | SetHipCircRadius !HipCirc !Double next
  | AddHipStaticPoly ![R2] next
  | RemoveHipCirc    !HipCirc next
  | SetGravity       !R2 next
  | SetDamping       !Double next
```

---

# GameM

See module `GameM`

```Haskell
data GameScript next =
    forall a. Random a => GetRandom    (a,a) (a -> next)
  | forall a.             EvalRand     (Rand StdGen a) (a -> next)
  |                       Get          (GameState -> next)
  |                       Modify       (GameState -> GameState) next
  |                       Put          !GameState next
  |                       PrintStr     !String next
  |                       GetTime      (UTCTime -> next)
  |                       TimeSince    UTCTime (Double -> next)
  | forall a.             RunGLM       (GLM a) (a -> next)
  |                       NewHipSpace  (HipSpace -> next)
  | forall a.             RunHipM      !HipSpace (HipM a) (a -> next)
```

---

# Exercises for Epidemic

1. Instead of having germs just divide in two, make them divide into a random
   number of germs

2. Make the germs "shirk away" from a death nearby. How quickly they move should
   be proportional to how close the death/tap was to them

3. Add gravity into the game. Create a beaker using a few `AddHipStaticPoly`s from the `HipM`
   monad

---

# Challenge Exercise

1. Create Pong from scratch using Helm
