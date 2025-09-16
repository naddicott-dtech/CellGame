pseudocode.md
Overview
Director owns the Flash stage, sets up a global state machine, and delegates to subsystems such as the Engine, menus, and cinematics.

Engine drives in-level play: it loads baked XML level data, creates the world, hooks up the cell simulation, and processes frame updates, resources, objectives, and faux-pause events.

World holds the rendered level, cell, and overlays, while WorldCanvas tracks off-cell entities (goodies, toxins, etc.) around the playable lens and relays resource popups.

The Cell encapsulates organelles, resources, membrane, and lists of active entities; CellObject is the shared base for anything selectable inside the cell.

Level scripting relies on baked XML parsed by BakedLevel and runtime orchestration by WizardOfOz, which manages spawn batches, waves, and goodies.

Interface drives HUD widgets, tooltips, zoom, and action menus for the player.

The membrane implements a spring-node perimeter with grav points, forming the soft-body behavior to recreate in Godot’s SoftBody2D system.

Bootstrapping & State Control (Director)
function Director.siteLock():
    connectToApis()
    domain = extractDomain(stage.loaderInfo.url)
    if SITE_LOCK enabled and domain not allowed: abort startup
    else: init()
#​:codex-file-citation[codex-file-citation]{line_range_start=224 line_range_end=248 path=src/Director.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Director.as#L224-L248"}​

function Director.init():
    state = new DState()
    startItUp()
    listen to ENTER_FRAME via run()
#​:codex-file-citation[codex-file-citation]{line_range_start=370 line_range_end=379 path=src/Director.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Director.as#L370-L379"}​

function Director.startItUp():
    LevelProgress.initProgress()
    makeSoundMgr()
    makePauseSprite()
    initListeners()        # keyboard, mouse, wheel
    loadGameLevelInfo()
    showCinema(Cinema.SPLASH)
#​:codex-file-citation[codex-file-citation]{line_range_start=382 line_range_end=422 path=src/Director.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Director.as#L382-L422"}​

function Director.showTitle():
    stage.quality = StageQuality.HIGH
    newState(DState.TITLE)
    makeTitle()
#​:codex-file-citation[codex-file-citation]{line_range_start=432 line_range_end=455 path=src/Director.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Director.as#L432-L455"}​

function Director.goPlayGame():
    hideTitle()
    newState(DState.INGAME)
    showMenu(MenuSystem.LEVELPICKER)
#​:codex-file-citation[codex-file-citation]{line_range_start=465 line_range_end=475 path=src/Director.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Director.as#L465-L475"}​

function Director.makeEngine():
    assert c_engine is null
    c_engine = new Engine()
    addChild(c_engine)
    c_engine.setDirector(this)
    c_engine.init(curr_level)
    makeCursor()
#​:codex-file-citation[codex-file-citation]{line_range_start=486 line_range_end=498 path=src/Director.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Director.as#L486-L498"}​

function Director.quit/reset/end level:
    tear down c_engine, update stats, revert state stack or show menus as needed
#​:codex-file-citation[codex-file-citation]{line_range_start=499 line_range_end=600 path=src/Director.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Director.as#L499-L600"}​
Engine Setup & Game Loop
function Engine.init(level_index):
    Log.begin_level(level_index)
    setupFlags()
    setupLists()                  # populate action lists per organelle type
    setupObjectiveHandler()
    loadLevel(level_index)        # choose embedded XML class
    c_interface.init()
    c_interface.setDirector(p_director)
    p_virusGlass = c_interface.c_tutorialGlass.c_virusGlass
    flushInterface()
    makeListeners()
    makeSelecter()
    makeActionMenu()
    setupCursor()
    initMessaging()
    hackTimer = Timer(HACK_TIME, 3)
#​:codex-file-citation[codex-file-citation]{line_range_start=305 line_range_end=334 path=src/Engine.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Engine.as#L305-L334"}​​:codex-file-citation[codex-file-citation]{line_range_start=448 line_range_end=487 path=src/Engine.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Engine.as#L448-L487"}​

function Engine.onLevelLoad():
    stage.quality = MenuSystem_InGame._quality
    lvl.levelData.level_index = theLevelNum
    getStartResources()
    Director.startMusic(MUS_CALM, loop=true)
    makeWorld()
    makeWOZ()
    makeTutArchive()
    c_interface.setZoomScale(0.05)
    c_interface.setDiscovery(0)
    c_interface.setDiscoveryMax(levelData.levelDiscoveries.length)
    loadObjectives()
    play intro masker once
    p_cell.setCytoProcess(true)
#​:codex-file-citation[codex-file-citation]{line_range_start=368 line_range_end=399 path=src/Engine.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Engine.as#L368-L399"}​

function Engine.makeWorld():
    c_world = new World()
    addChild(c_world)
    wire director, engine, interface references
    apply level background, size, start point, boundary
    c_world.init()
    keep selecter/interface on top
    listen to RUNFRAME and FAUXFRAME events
    p_cursor.setWorld(c_world)
#​:codex-file-citation[codex-file-citation]{line_range_start=910 line_range_end=935 path=src/Engine.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Engine.as#L910-L935"}​

function Engine.makeWOZ():
    d_woz = new WizardOfOz()
    link engine, cell, canvas
    d_woz.init(levelData, c_world.getMaskSize())
#​:codex-file-citation[codex-file-citation]{line_range_start=943 line_range_end=952 path=src/Engine.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Engine.as#L943-L952"}​

function Engine.run(frame_event):
    tick()                        # transactions, input, messaging, HUD flush
    c_world.dispatchEvent(frame_event)
    c_interface.dispatchEvent(frame_event)
    d_woz.dispatchEvent(frame_event)
    oHandler.dispatchEvent(frame_event)
#​:codex-file-citation[codex-file-citation]{line_range_start=735 line_range_end=751 path=src/Engine.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Engine.as#L735-L751"}​

function Engine.fauxRun():
    drive faux-pause commands, then call p_cell.fauxRun() to keep membrane/vesicles visually updated
#​:codex-file-citation[codex-file-citation]{line_range_start=700 line_range_end=719 path=src/Engine.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Engine.as#L700-L719"}​
World & Camera Structure
class World extends Sprite:
    init():
        scrollPoint = Point(0,0)
        position world center at stage center
        makeTerrain(backgroundIndex, levelWidth, levelHeight)
        makeObjectLayer()         # adds WorldCanvas, mask, cell
        makeMask()
        makeCell()
        goStartPoint()
        add MOUSE_DOWN listener -> engine.onWorldMouseDown()
        add RUNFRAME listener -> run()
#​:codex-file-citation[codex-file-citation]{line_range_start=1 line_range_end=149 path=src/World.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/World.as#L1-L149"}​

function World.goStartPoint():
    c_cell.moveCellTo(startX * Terrain.SCALE_MULT, startY * Terrain.SCALE_MULT)
    centerOnCell()
#​:codex-file-citation[codex-file-citation]{line_range_start=116 line_range_end=119 path=src/World.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/World.as#L116-L119"}​

function World.run(RunFrameEvent):
    c_cell.dispatchEvent(event)
    c_canvas.dispatchEvent(event)
#​:codex-file-citation[codex-file-citation]{line_range_start=144 line_range_end=147 path=src/World.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/World.as#L144-L147"}​

class WorldCanvas extends GameObject:
    init():
        create lists for running objects and goodies
        c_cgrid = ObjectGrid()
        listen to RUNFRAME
#​:codex-file-citation[codex-file-citation]{line_range_start=1 line_range_end=82 path=src/WorldCanvas.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/WorldCanvas.as#L1-L82"}​

    onCellMove(delta):
        update lens center to cell centrosome
        check lens bounds
        reposition goodies and floating popups
#​:codex-file-citation[codex-file-citation]{line_range_start=127 line_range_end=138 path=src/WorldCanvas.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/WorldCanvas.as#L127-L138"}​

    showMeTheMoneyArray(resources, x, y):
        render popup icons and forward resource production to engine
#​:codex-file-citation[codex-file-citation]{line_range_start=147 line_range_end=179 path=src/WorldCanvas.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/WorldCanvas.as#L147-L179"}​
Cell Simulation Core
class Cell extends GameObject:
    data fields:
        current/max resources (ATP, NA, AA, FA, G)
        toxin level, pH, cytoplasm volume
        organelle lists (ribosomes, lysosomes, peroxisomes, slicers, mitochondria, chloroplasts)
        running/selectable objects, viruses, vesicles
        object grid and timers for clearing, production, necrosis
#​:codex-file-citation[codex-file-citation]{line_range_start=21 line_range_end=188 path=src/Cell.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Cell.as#L21-L188"}​

    init():
        setCentLoc(0,0)
        makeObjectGrid()
        setChildren()             # nucleus, centrosome, ER, membrane, Golgi, skeleton
        makeLists()
        listen to RUNFRAME and MOUSE_DOWN
#​:codex-file-citation[codex-file-citation]{line_range_start=181 line_range_end=188 path=src/Cell.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Cell.as#L181-L188"}​

    run(RunFrameEvent):
        for each list_running entry: dispatch event, prune nulls
        clearTick()               # refresh object grid & membrane nodes periodically
        flush()                   # sync selections/resources to interface
#​:codex-file-citation[codex-file-citation]{line_range_start=247 line_range_end=324 path=src/Cell.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Cell.as#L247-L324"}​

    fauxRun():
        c_membrane.drawAllNodes()
        update vesicle visuals and virus-held vesicles
#​:codex-file-citation[codex-file-citation]{line_range_start=234 line_range_end=245 path=src/Cell.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Cell.as#L234-L245"}​

    selectOne/ selectMany(pass to engine), setSelectType(for selection overlay)
#​:codex-file-citation[codex-file-citation]{line_range_start=1010 line_range_end=1032 path=src/Cell.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Cell.as#L1010-L1032"}​

    spend/spendATP/produce/rewardProduce delegate to Engine, optionally show popups via WorldCanvas
#​:codex-file-citation[codex-file-citation]{line_range_start=1080 line_range_end=1166 path=src/Cell.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Cell.as#L1080-L1166"}​

    organelle placement helpers (placeMitochondrion/Chloroplast):
        instantiate organelle
        position with cytoskeleton guidance
        optionally flag as outside cell or damaged
#​:codex-file-citation[codex-file-citation]{line_range_start=1291 line_range_end=1346 path=src/Cell.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Cell.as#L1291-L1346"}​
Membrane & Soft-Body Behavior
Membrane is a CellObject with lists of nodes, grav points, and tracked viruses/basic units; it draws spring, gap, and outline shapes to visualize the soft boundary.

Node management uses stretch constants, health per node, and handles drag/pseudopod states—mirror these with Godot SoftBody2D by mapping nodes to physics points, maintaining spring connections, and updating visuals each frame as Cell.fauxRun() does today.

Selection & Organelle Objects
class CellObject extends Selectable:
    holds reference to parent Cell and optional Microtubule
    compute movement costs (ATP burn per micron)
    overrides mouseDown/click to inform Cell/Engine selection
    moveToPoint uses engine action lists fetched from cell
    supports being in vesicles, outside cell, or colliding with membrane
    handles cytoplasm deployment, plop sounds, and animation toggles
#​:codex-file-citation[codex-file-citation]{line_range_start=14 line_range_end=200 path=src/CellObject.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/CellObject.as#L14-L200"}​

function CellObject.receiveActionList():
    list_actions = cell.getActionListFromEngine(num_id)
#​:codex-file-citation[codex-file-citation]{line_range_start=79 line_range_end=85 path=src/CellObject.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/CellObject.as#L79-L85"}​
Resource Economy
function Engine.canAfford(atp, na, aa, fa, g):
    compare requested costs to stored resources
    record which resource blocked the transaction
#​:codex-file-citation[codex-file-citation]{line_range_start=3434 line_range_end=3454 path=src/Engine.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Engine.as#L3434-L3454"}​

function Engine.spend(costArray[ATP, NA, AA, FA, G]):
    if all resources available:
        subtract, notify ObjectiveHandler of resource change, mark dirty
        return true
    else: return false
#​:codex-file-citation[codex-file-citation]{line_range_start=3566 line_range_end=3597 path=src/Engine.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Engine.as#L3566-L3597"}​

function Engine.produce(deltaArray):
    add resources, clamp to max, notify handlers, mark dirty
#​:codex-file-citation[codex-file-citation]{line_range_start=3502 line_range_end=3515 path=src/Engine.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Engine.as#L3502-L3515"}​

function Engine.loseResources(deltaArray):
    subtract but clamp at zero, notify handlers
#​:codex-file-citation[codex-file-citation]{line_range_start=3532 line_range_end=3549 path=src/Engine.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Engine.as#L3532-L3549"}​

function Cell.rewardProduce(resources, multiplier, popupPoint):
    scale array
    WorldCanvas.justShowMeTheMoneyArray for feedback (no spend)
    Engine.produce(resources)
#​:codex-file-citation[codex-file-citation]{line_range_start=1138 line_range_end=1166 path=src/Cell.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Cell.as#L1138-L1166"}​
Level Data & Wave Management
class BakedLevel(e:Engine):
    bakeData():
        parse embedded XML levelInfo into LevelData
        capture start resources, organelle counts, boundary info
        build StuffEntry lists for goodies/objects/enemies
        build ThingEntry lists for placed entities and waves (id, type, count, delay, spread, vesicle)
        instantiate objectives with targets, triggers, tutorials
#​:codex-file-citation[codex-file-citation]{line_range_start=1 line_range_end=200 path=src/com/pecLevel/BakedLevel.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/com/pecLevel/BakedLevel.as#L1-L200"}​

class WizardOfOz extends EventDispatcher:
    init(levelData, lensSize):
        create GoodieData batches for resources
        size world via Terrain scale
        populate virus/object lists and waves from StuffEntry/ThingEntry
        update lens radius constants, listen to RUNFRAME
#​:codex-file-citation[codex-file-citation]{line_range_start=1 line_range_end=168 path=src/com/woz/WizardOfOz.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/com/woz/WizardOfOz.as#L1-L168"}​

    tickTime():
        periodically spawn goodies/batches, manage wave timers, interact with Engine/Cell via events
#​:codex-file-citation[codex-file-citation]{line_range_start=147 line_range_end=166 path=src/com/woz/WizardOfOz.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/com/woz/WizardOfOz.as#L147-L166"}​​:codex-file-citation[codex-file-citation]{line_range_start=943 line_range_end=952 path=src/Engine.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Engine.as#L943-L952"}​
UI Overlay
class Interface extends Sprite:
    init():
        initialize zoomer, quant panel, message bar
        trap mouse events so clicks don't leak to game world
        listen to RUNFRAME to update resource meters and selected panel
        construct ActionMenu
#​:codex-file-citation[codex-file-citation]{line_range_start=1 line_range_end=120 path=src/Interface.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Interface.as#L1-L120"}​

    setupButtons():
        pause/menu toggles
        follow button toggles camera follow state
#​:codex-file-citation[codex-file-citation]{line_range_start=142 line_range_end=150 path=src/Interface.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Interface.as#L142-L150"}​

    showTooltip/showMessage/showAlert/showSpeech manage HUD text and tutorial overlays
#​:codex-file-citation[codex-file-citation]{line_range_start=157 line_range_end=196 path=src/Interface.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Interface.as#L157-L196"}​

    onClickNewThingMenu(name):
        dismiss popup and delegate to Engine.onClickNewThing
#​:codex-file-citation[codex-file-citation]{line_range_start=157 line_range_end=169 path=src/Interface.as git_url="https://github.com/naddicott-dtech/CellGame/blob/master/src/Interface.as#L157-L169"}​
