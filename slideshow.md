# Writing games for Android in Haskell

## Sean Seefried & Christian Marie


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
## Processing the "Playing Level" state

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




# Co-ordinate systems

* back-end coordinates
* world co-ordinates


# Exercises

1. Make the germs "shirk away" from a death nearby. How quickly they move should
   be proportional to how close the death/tap was to them

2. Instead of having germs just divide in two, make them divide into a random
   number of germs

3. Bug-fix. You can select and drag around a germ. Occasionally you "pick up" another
   germ and lose the first. Have a look at `playingLevelDrag` and see why

