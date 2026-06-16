repeat task.wait() until game:IsLoaded()
-- ============================================================
-- Radiant Hub v6 fix53
-- ============================================================

-- WYNF_NO_VIRTUALIZE stub (stripped automatically in obfuscated builds)
if not WYNF_OBFUSCATED then
	WYNF_NO_VIRTUALIZE = function(f) return f end
end

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local ContentProvider = game:GetService("ContentProvider")
local Stats = game:GetService("Stats")
local LP = Players.LocalPlayer

task.spawn(function() end)

local _isfile = isfile or (syn and syn.isfile) or (getgenv and getgenv().isfile) or function() return false end
local _readfile = readfile or (syn and syn.readfile) or (getgenv and getgenv().readfile) or function() return nil end
local _writefile= writefile or (syn and syn.writefile) or (getgenv and getgenv().writefile) or function() end
local getconnections = getconnections or get_signal_cons or getconnects or (syn and syn.get_signal_cons)

-- ============================================================
-- STATE
-- ============================================================
local State = {
normalSpeed=60, carrySpeed=30, laggerSpeed=10.1,
speedToggled=false, laggerEnabled=false,
infJumpEnabled=false, antiRagdollEnabled=false, noAnimEnabled=false,
guiVisible=true, uiLocked=false, guiMinimized=false,
isStealing=false, stealStartTime=nil, lastStealTick=0,
autoLeftEnabled=false, autoRightEnabled=false,
autoLeftPhase=1, autoRightPhase=1,
medusaLastUsed=0, medusaDebounce=false, medusaCounterEnabled=false,
batAimbotToggled=false, autoSwingEnabled=false,
batCounterEnabled=false, batCounterDebounce=false,
dropEnabled=false, _tpInProgress=false,
stretchRezEnabled=false, stretchRezFov=120, nightTimeEnabled=false, purpleSkyEnabled=false, fpsBoosterEnabled=false,
autoTpDownEnabled=false, autoTpDownHeight=20,
lastMoveDir=Vector3.new(0,0,0),
stackButtonsHidden=false,
_prevCarry=30, _prevSpeed=false,
duelCountdownEnabled=false, _duelWaiting=false,
espEnabled=false,
baseEspEnabled=false,
animEnabled=false,
_autoPlaySpeed=50,
}

local Keys = {
speed=Enum.KeyCode.Q, guiHide=Enum.KeyCode.LeftControl,
autoLeft=Enum.KeyCode.L, autoRight=Enum.KeyCode.R,
lagger=Enum.KeyCode.Unknown, tpDown=Enum.KeyCode.Unknown,
drop=Enum.KeyCode.H, aimbot=Enum.KeyCode.Unknown,
}

-- ============================================================
-- DEFAULT STACK BUTTON POSITIONS (for reset)
-- ============================================================
local BTN_W=73; local BTN_H=65; local BTN_GAP=10; local COLS=2
local stackDefs = {
{key="autoLeft", label="AUTO\nLEFT"},
{key="autoRight", label="AUTO\nRIGHT"},
{key="aimbot", label="AIMBOT"},
{key="lagger", label="LAGGER\nMODE"},
{key="drop", label="DROP\nBR"},
{key="tpDown", label="TP\nDOWN"},
{key="carrySpeed", label="CARRY\nSPEED"},
{key="taunt", label="TAUNT", noToggle=true},
}
local GRID_W=COLS*(BTN_W+BTN_GAP)-BTN_GAP
local GRID_H=math.ceil(#stackDefs/COLS)*(BTN_H+BTN_GAP)-BTN_GAP

local getDefaultStackPos = WYNF_NO_VIRTUALIZE(function(i)
local col=(i-1)%COLS
local row2=math.floor((i-1)/COLS)
return UDim2.new(1,-(GRID_W+14)+col*(BTN_W+BTN_GAP),0.5,-(GRID_H/2)+row2*(BTN_H+BTN_GAP))
end)

local Steal = {
AutoStealEnabled=false, StealRadius=20, StealDuration=0.25,
Data={}, plotCache={}, plotCacheTime={}, cachedPrompts={}, promptCacheTime=0,
}

-- ============================================================
-- PRESETS
-- ============================================================
local Presets = {}
local PRESET_FILE = "RadiantHubPresets.json"
local LAST_PRESET_FILE = "RadiantHubLastPreset.json"
local CONFIG_FILE = "RadiantHubConfig.json"

local function buildPresetSnapshot()
return {
normalSpeed = State.normalSpeed,
carrySpeed = State.carrySpeed,
laggerSpeed = State.laggerSpeed,
stealRadius = Steal.StealRadius,
stealDuration = Steal.StealDuration,
infJump = State.infJumpEnabled,
antiRagdoll = State.antiRagdollEnabled,
medusaCounter = State.medusaCounterEnabled,
batCounter = State.batCounterEnabled,
autoSteal = Steal.AutoStealEnabled,
}
end

local function savePresetsFile()
local ok,encoded=pcall(function() return HttpService:JSONEncode(Presets) end)
if ok then pcall(function() _writefile(PRESET_FILE,encoded) end) end
end

local function loadPresetsFile()
local hasFile=false; pcall(function() hasFile=_isfile(PRESET_FILE) end)
if not hasFile then return end
local raw; pcall(function() raw=_readfile(PRESET_FILE) end)
if not raw then return end
local ok,decoded=pcall(function() return HttpService:JSONDecode(raw) end)
if ok and decoded then Presets=decoded end
end

local function saveLastPresetName(name)
local ok, encoded = pcall(function() return HttpService:JSONEncode({lastPreset=name}) end)
if ok then pcall(function() _writefile(LAST_PRESET_FILE, encoded) end) end
end

local function loadLastPresetName()
local hasFile = false; pcall(function() hasFile = _isfile(LAST_PRESET_FILE) end)
if not hasFile then return nil end
local raw; pcall(function() raw = _readfile(LAST_PRESET_FILE) end)
if not raw then return nil end
local ok, decoded = pcall(function() return HttpService:JSONDecode(raw) end)
if ok and decoded then return decoded.lastPreset end
return nil
end

local MOVE_KEYS={[Enum.KeyCode.W]=true,[Enum.KeyCode.A]=true,[Enum.KeyCode.S]=true,[Enum.KeyCode.D]=true,
[Enum.KeyCode.Up]=true,[Enum.KeyCode.Left]=true,[Enum.KeyCode.Down]=true,[Enum.KeyCode.Right]=true}

local PLOT_CACHE_DURATION=2; local PROMPT_CACHE_REFRESH=0.15
local STEAL_COOLDOWN=0.1; local MEDUSA_COOLDOWN=25; local DROP_AUTO_OFF_DELAY=0.15

local POS={
L1=Vector3.new(-476.48,-6.28,92.73), L2=Vector3.new(-483.12,-4.95,94.80),
R1=Vector3.new(-476.16,-6.52,25.62), R2=Vector3.new(-483.04,-5.09,23.14),
}

local Conns={autoSteal=nil,antiRag=nil,autoLeft=nil,autoRight=nil,aimbot=nil,anchor={},progress=nil,batCounter=nil}
local modeToggleState = "full" -- "full" = Full Auto Play, "semi" = Semi Auto Play

local h,hrp
local setAutoLeft,setAutoRight,setInfJump,setAntiRag,setNoAnim
local setMedusaCounter,setAimbot,setAutoSwing
local setLagger,setDropBrainrot,setInstaGrab
local startNoAnim, stopNoAnim
local setupChar
local _billLbl = nil
local _billLastChar = nil
local setupMedusaCounter,stopMedusaCounter,startAntiRagdoll,stopAntiRagdoll
local startAutoSteal,stopAutoSteal
local startAutoLeft,stopAutoLeft,startAutoRight,stopAutoRight
local saveConfig,loadConfig,runDropBrainrot,stopDropBrainrot,doTpDown
local startBatCounter,stopBatCounter,setBatCounter
local stackBtnRefs={}; local stackWrappers={}; local keybindBtnRefs={}
local normalBox,carryBox,laggerBox,uiScaleBox,stealRadBox,lockBtn
local setLockUI, stealDurBox
local setStretchRez, setNightTime, setPurpleSky, setFpsBooster
local stretchRezFovBox
local setPlayerESP, setBaseESP
local setTryhardAnim
local _espConns = {}
local _baseESPOriginals = {}
local setAutoTpDown
local autoTpDownHeightBox
local setHideButtonsToggle
local radTB
local modeBtn

local presetListFrame=nil
local presetNameBox=nil
local rebuildPresetList

-- ============================================================
-- COLORS (BLACK THEME)
-- ============================================================
-- RADIENT HUB THEME
local RED    = Color3.fromRGB(57, 255, 20)
local RED_DIM= Color3.fromRGB(20, 80, 20)
local C = {
winBg       = Color3.fromRGB(18, 18, 22),
winBorder   = Color3.fromRGB(42, 42, 53),
topBg       = Color3.fromRGB(26, 26, 34),
topTitle    = Color3.fromRGB(57, 255, 20),  -- green title (Radiant Hub)
topSub      = Color3.fromRGB(160, 160, 176),
topBtn      = Color3.fromRGB(26, 26, 34),
topBtnHov   = Color3.fromRGB(20, 80, 20),
topDivider  = Color3.fromRGB(42, 42, 53),    -- dark divider line under title
tabBarBg    = Color3.fromRGB(26, 26, 34),
tabBarDiv   = Color3.fromRGB(42, 42, 53),
tabIdle     = Color3.fromRGB(160, 160, 176),
tabActive   = Color3.fromRGB(57, 255, 20),
tabActiveBg = Color3.fromRGB(26, 26, 34),
tabUnderline= Color3.fromRGB(57, 255, 20),
sectionTxt  = Color3.fromRGB(57, 255, 20),  -- green section labels
sectionDiv  = Color3.fromRGB(42, 42, 53),
rowBg       = Color3.fromRGB(26, 26, 34),
rowBorder   = Color3.fromRGB(42, 42, 53),
rowLabel    = Color3.fromRGB(237, 237, 237),
rowSub      = Color3.fromRGB(160, 160, 176),
rowValue    = Color3.fromRGB(237, 237, 237),
rowHov      = Color3.fromRGB(42, 42, 53),
inputBg     = Color3.fromRGB(26, 26, 34),
inputBorder = Color3.fromRGB(42, 42, 53),
inputFocus  = Color3.fromRGB(57, 255, 20),
inputTxt    = Color3.fromRGB(237, 237, 237),
pillOff     = Color3.fromRGB(18, 18, 22),
pillOn      = Color3.fromRGB(57, 255, 20),
dotOff      = Color3.fromRGB(100, 100, 120),
dotOn       = Color3.fromRGB(255, 255, 255),
pillBorder  = Color3.fromRGB(42, 42, 53),
modeBtnBg   = Color3.fromRGB(26, 26, 34),
modeBtnBrd  = Color3.fromRGB(42, 42, 53),
modeBtnTxt  = Color3.fromRGB(200, 200, 210),
modeBtnActBg= Color3.fromRGB(57, 255, 20),
modeBtnActTx= Color3.fromRGB(255, 255, 255),
chipBg      = Color3.fromRGB(26, 26, 34),
chipBorder  = Color3.fromRGB(42, 42, 53),
chipTxt     = Color3.fromRGB(220, 220, 230),
btnBg       = Color3.fromRGB(26, 26, 34),
btnBorder   = Color3.fromRGB(42, 42, 53),
btnTxt      = Color3.fromRGB(237, 237, 237),
btnHov      = Color3.fromRGB(42, 42, 53),
stackBg     = Color3.fromRGB(18, 18, 22),
stackBrd    = Color3.fromRGB(42, 42, 53),
stackTxt    = Color3.fromRGB(255, 255, 255),
stackActBg  = Color3.fromRGB(26, 26, 34),
stackActBrd = Color3.fromRGB(57, 255, 20),
stackActTxt = Color3.fromRGB(255, 255, 255),
stackDot    = Color3.fromRGB(18, 18, 22),
stackDotOn  = Color3.fromRGB(57, 255, 20),
infoBg      = Color3.fromRGB(26, 26, 34),
infoBrd     = Color3.fromRGB(42, 42, 53),
infoTxt     = Color3.fromRGB(160, 160, 176),
infoVal     = Color3.fromRGB(237, 237, 237),
infoFill    = Color3.fromRGB(57, 255, 20),
accent      = Color3.fromRGB(42, 42, 53),
accentDim   = Color3.fromRGB(26, 26, 34),
presetBg    = Color3.fromRGB(26, 26, 34),
presetBrd   = Color3.fromRGB(42, 42, 53),
presetLoad  = Color3.fromRGB(57, 255, 20),
presetDel   = Color3.fromRGB(180, 60, 180),
delBrd      = Color3.fromRGB(120, 40, 140),
lockOn      = Color3.fromRGB(57, 255, 20),
divider     = Color3.fromRGB(42, 42, 53),
}

-- ============================================================
-- CLEANUP
-- ============================================================
for _,name in pairs({"VyseSlottedGUI","VyseAsireGUI","VyseAsireHubV4","VyseAsireHubV5","VyseAsireHubV5_1","AsireHubV5_1","AsireHubV5_2","OpiumGGV5_2","RadiantHubV1","RadiantHubInfoBar"}) do
pcall(function() local o=game:GetService("CoreGui"):FindFirstChild(name); if o then o:Destroy() end end)
pcall(function() local o=LP:WaitForChild("PlayerGui"):FindFirstChild(name); if o then o:Destroy() end end)
end

-- ============================================================
-- ROOT GUI
-- ============================================================
-- ============================================================
-- TRYHARD ANIM SYSTEM (from Senerity)
-- ============================================================
local Anims = {
    idle1="rbxassetid://133806214992291", idle2="rbxassetid://94970088341563",
    walk="rbxassetid://707897309",        run="rbxassetid://707861613",
    jump="rbxassetid://116936326516985",  fall="rbxassetid://116936326516985",
    climb="rbxassetid://116936326516985", swim="rbxassetid://116936326516985",
    swimidle="rbxassetid://116936326516985",
}
local animHeartbeatConn, originalAnims

local function isPackAnim(id)
    if not id then return false end
    for _,v in pairs(Anims) do if v==id then return true end end
    return false
end
local function saveOriginalAnims(char)
    local anim=char:FindFirstChild("Animate"); if not anim then return end
    local function g(o) return o and o.AnimationId or nil end
    local ids={
        idle1=g(anim.idle and anim.idle.Animation1), idle2=g(anim.idle and anim.idle.Animation2),
        walk=g(anim.walk and anim.walk.WalkAnim),    run=g(anim.run and anim.run.RunAnim),
        jump=g(anim.jump and anim.jump.JumpAnim),    fall=g(anim.fall and anim.fall.FallAnim),
        climb=g(anim.climb and anim.climb.ClimbAnim),swim=g(anim.swim and anim.swim.Swim),
        swimidle=g(anim.swimidle and anim.swimidle.SwimIdle),
    }
    if not isPackAnim(ids.walk) then originalAnims=ids end
end
local function applyAnimPack(char)
    local anim=char:FindFirstChild("Animate"); if not anim then return end
    local function s(o,id) if o then o.AnimationId=id end end
    s(anim.idle and anim.idle.Animation1,Anims.idle1); s(anim.idle and anim.idle.Animation2,Anims.idle2)
    s(anim.walk and anim.walk.WalkAnim,Anims.walk);   s(anim.run and anim.run.RunAnim,Anims.run)
    s(anim.jump and anim.jump.JumpAnim,Anims.jump);   s(anim.fall and anim.fall.FallAnim,Anims.fall)
    s(anim.climb and anim.climb.ClimbAnim,Anims.climb); s(anim.swim and anim.swim.Swim,Anims.swim)
    s(anim.swimidle and anim.swimidle.SwimIdle,Anims.swimidle)
end
local function restoreOriginalAnims(char)
    if not originalAnims then return end
    local anim=char:FindFirstChild("Animate"); if not anim then return end
    local function s(o,id) if o and id then o.AnimationId=id end end
    s(anim.idle and anim.idle.Animation1,originalAnims.idle1); s(anim.idle and anim.idle.Animation2,originalAnims.idle2)
    s(anim.walk and anim.walk.WalkAnim,originalAnims.walk);   s(anim.run and anim.run.RunAnim,originalAnims.run)
    s(anim.jump and anim.jump.JumpAnim,originalAnims.jump);   s(anim.fall and anim.fall.FallAnim,originalAnims.fall)
    s(anim.climb and anim.climb.ClimbAnim,originalAnims.climb); s(anim.swim and anim.swim.Swim,originalAnims.swim)
    s(anim.swimidle and anim.swimidle.SwimIdle,originalAnims.swimidle)
    local hum2=char:FindFirstChildOfClass("Humanoid")
    if hum2 then for _,t in ipairs(hum2:GetPlayingAnimationTracks()) do t:Stop(0) end; hum2:ChangeState(Enum.HumanoidStateType.Running) end
end
local function startAnimToggle()
    if animHeartbeatConn then animHeartbeatConn:Disconnect(); animHeartbeatConn=nil end
    local char=LP.Character
    if char then
        saveOriginalAnims(char); applyAnimPack(char)
        local hum2=char:FindFirstChildOfClass("Humanoid")
        if hum2 then for _,t in ipairs(hum2:GetPlayingAnimationTracks()) do t:Stop(0) end; hum2:ChangeState(Enum.HumanoidStateType.Running) end
    end
    animHeartbeatConn=RunService.Heartbeat:Connect(function()
        if not State.animEnabled then return end
        local c=LP.Character; if c then applyAnimPack(c) end
    end)
end
local function stopAnimToggle()
    if animHeartbeatConn then animHeartbeatConn:Disconnect(); animHeartbeatConn=nil end
    State.animEnabled=false
    local char=LP.Character; if char then restoreOriginalAnims(char) end
end

local gui=Instance.new("ScreenGui")
gui.Name="RadiantHubV1"; gui.ResetOnSpawn=false; gui.DisplayOrder=20
gui.IgnoreGuiInset=true; gui.ZIndexBehavior=Enum.ZIndexBehavior.Sibling
gui.Parent=LP:WaitForChild("PlayerGui")

local uiScaleObj=Instance.new("UIScale",gui); uiScaleObj.Scale=1.0

-- Watermark bar removed

-- ============================================================
-- HELPERS
-- ============================================================
local mkCorner = WYNF_NO_VIRTUALIZE(function(p,r) local c=Instance.new("UICorner",p); c.CornerRadius=UDim.new(0,r or 6); return c end)
local mkStroke = WYNF_NO_VIRTUALIZE(function(p,col,th)
local s=Instance.new("UIStroke",p); s.Color=col; s.Thickness=th or 1
s.ApplyStrokeMode=Enum.ApplyStrokeMode.Border; return s
end)
local mkPad = WYNF_NO_VIRTUALIZE(function(p,t,b,l,r)
local pd=Instance.new("UIPadding",p)
pd.PaddingTop=UDim.new(0,t or 0); pd.PaddingBottom=UDim.new(0,b or 0)
pd.PaddingLeft=UDim.new(0,l or 0); pd.PaddingRight=UDim.new(0,r or 0)
return pd
end)

local function addGradientOutline(frame)
-- Static gradient only — no per-frame loop (removed animated offset which burned RenderStepped every frame)
local gradient = Instance.new("UIGradient", frame)
gradient.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0, Color3.fromRGB(60, 60, 60)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(200, 180, 100)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(60, 60, 60))
}
gradient.Rotation = 90
return gradient
end

-- ============================================================
-- DRAG
-- ============================================================
local function makeDraggable(frame,handle)
local src=handle or frame
local dragging,dragInput,dragStart,startAbsPos=false,nil,nil,nil
src.InputBegan:Connect(function(inp)
if State.uiLocked then return end
if inp.UserInputType==Enum.UserInputType.MouseButton1 or inp.UserInputType==Enum.UserInputType.Touch then
dragging=true; dragStart=inp.Position; startAbsPos=frame.AbsolutePosition
inp.Changed:Connect(function() if inp.UserInputState==Enum.UserInputState.End then dragging=false end end)
end
end)
src.InputChanged:Connect(function(inp)
if inp.UserInputType==Enum.UserInputType.MouseMovement or inp.UserInputType==Enum.UserInputType.Touch then dragInput=inp end
end)
UIS.InputChanged:Connect(function(inp)
if inp==dragInput and dragging and not State.uiLocked then
local dx=inp.Position.X-dragStart.X; local dy=inp.Position.Y-dragStart.Y
local sw=workspace.CurrentCamera.ViewportSize.X
local sh=workspace.CurrentCamera.ViewportSize.Y
local fw=frame.AbsoluteSize.X; local fh=frame.AbsoluteSize.Y
local newX=math.clamp(startAbsPos.X+dx, 0, sw-fw)
local newY=math.clamp(startAbsPos.Y+dy, 0, sh-fh)
frame.Position=UDim2.new(0, newX, 0, newY)
end
end)
end

-- Per-button drag: each button drags independently
local function makeStackDraggable(frame, onTap)
local _dragging = false
local _dragInput = nil
local _dragStart = nil
local _startPos = nil
local _moved = false

frame.InputBegan:Connect(function(inp)
    if inp.UserInputType ~= Enum.UserInputType.MouseButton1 and inp.UserInputType ~= Enum.UserInputType.Touch then return end
    if State.uiLocked then
        inp.Changed:Connect(function()
            if inp.UserInputState == Enum.UserInputState.End then
                if onTap then onTap() end
            end
        end)
        return
    end
    _dragging = true
    _moved = false
    _dragStart = inp.Position
    _startPos = frame.Position
    inp.Changed:Connect(function()
        if inp.UserInputState == Enum.UserInputState.End then
            if not _moved and onTap then onTap() end
            if _moved then task.spawn(function() pcall(saveConfig) end) end
            _dragging = false
            _moved = false
        end
    end)
end)

frame.InputChanged:Connect(function(inp)
    if inp.UserInputType == Enum.UserInputType.MouseMovement or inp.UserInputType == Enum.UserInputType.Touch then
        _dragInput = inp
    end
end)

UIS.InputChanged:Connect(function(inp)
    if inp ~= _dragInput or not _dragging then return end
    if State.uiLocked then _dragging = false; _moved = false; return end
    local dx = inp.Position.X - _dragStart.X
    local dy = inp.Position.Y - _dragStart.Y
    if math.abs(dx) > 4 or math.abs(dy) > 4 then _moved = true end
    if _moved then
        frame.Position = UDim2.new(_startPos.X.Scale, _startPos.X.Offset + dx, _startPos.Y.Scale, _startPos.Y.Offset + dy)
    end
end)
end

-- ============================================================
-- MAIN WINDOW
-- ============================================================
local WIN_W = 460
local WIN_H = 440
local TITLE_H = 38
local TAB_H = 34
local SIDEBAR_W = 100

local mainOuter = Instance.new("Frame", gui)
mainOuter.Name = "MainOuter"
mainOuter.Size = UDim2.new(0, WIN_W, 0, WIN_H)
mainOuter.Position = UDim2.new(0.5, -WIN_W/2, 0.5, -WIN_H/2)
mainOuter.BackgroundColor3 = C.winBg
mainOuter.BackgroundTransparency = 0
mainOuter.BorderSizePixel = 0
mainOuter.ClipsDescendants = true
mainOuter.ZIndex = 10
mkCorner(mainOuter, 8)
local mainStroke = mkStroke(mainOuter, C.winBorder, 1.5)
-- Make main window draggable (drag by titleBar)
-- makeDraggable(mainOuter)

-- ============================================================
-- TITLE BAR (Aphex Hub style — logo left, title + sub, minimize/close right)
-- ============================================================
local titleBar = Instance.new("Frame", mainOuter)
titleBar.Size = UDim2.new(1, 0, 0, TITLE_H)
titleBar.BackgroundColor3 = C.topBg
titleBar.BackgroundTransparency = 0
titleBar.BorderSizePixel = 0
titleBar.ZIndex = 5
makeDraggable(mainOuter, titleBar)

-- Left accent bar (green vertical stripe like Aphex Hub)
local titleAccent = Instance.new("Frame", titleBar)
titleAccent.Size = UDim2.new(0, 3, 1, 0)
titleAccent.Position = UDim2.new(0, 0, 0, 0)
titleAccent.BackgroundColor3 = C.topTitle
titleAccent.BorderSizePixel = 0
titleAccent.ZIndex = 6

-- Title text "RADIANT HUB"
local titleLbl = Instance.new("TextLabel", titleBar)
titleLbl.Size = UDim2.new(0, 180, 0, 18)
titleLbl.Position = UDim2.new(0, 12, 0, 6)
titleLbl.BackgroundTransparency = 1
titleLbl.Text = "RADIANT HUB"
titleLbl.TextColor3 = C.topTitle
titleLbl.Font = Enum.Font.GothamBlack
titleLbl.TextSize = 14
titleLbl.TextXAlignment = Enum.TextXAlignment.Left
titleLbl.TextStrokeTransparency = 1
titleLbl.ZIndex = 6

-- Sub-label (discord style like Aphex Hub)
local titleSub = Instance.new("TextLabel", titleBar)
titleSub.Size = UDim2.new(0, 200, 0, 12)
titleSub.Position = UDim2.new(0, 12, 0, 24)
titleSub.BackgroundTransparency = 1
titleSub.Text = "discord.gg/RjvhQVzKC"
titleSub.TextColor3 = C.topSub
titleSub.Font = Enum.Font.Gotham
titleSub.TextSize = 10
titleSub.TextXAlignment = Enum.TextXAlignment.Left
titleSub.ZIndex = 6

-- CLOSE BUTTON (Aphex Hub style — small square "-" button top right)
local closeBtn = Instance.new("TextButton", titleBar)
closeBtn.Size = UDim2.new(0, 20, 0, 20)
closeBtn.Position = UDim2.new(1, -26, 0.5, -10)
closeBtn.BackgroundColor3 = C.modeBtnBg
closeBtn.BorderSizePixel = 0
closeBtn.Text = "−"
closeBtn.TextColor3 = C.topSub
closeBtn.Font = Enum.Font.GothamBlack
closeBtn.TextSize = 14
closeBtn.ZIndex = 7
mkCorner(closeBtn, 3)
mkStroke(closeBtn, C.chipBorder, 1)
closeBtn.MouseEnter:Connect(function() TweenService:Create(closeBtn, TweenInfo.new(0.1), {TextColor3 = Color3.fromRGB(220,80,80), BackgroundColor3 = Color3.fromRGB(60,20,20)}):Play() end)
closeBtn.MouseLeave:Connect(function() TweenService:Create(closeBtn, TweenInfo.new(0.1), {TextColor3 = C.topSub, BackgroundColor3 = C.modeBtnBg}):Play() end)
closeBtn.MouseButton1Click:Connect(function() State.guiVisible = false; mainOuter.Visible = false end)

-- MINIMIZE BUTTON
local minimizeBtn = Instance.new("TextButton", titleBar)
minimizeBtn.Size = UDim2.new(0, 20, 0, 20)
minimizeBtn.Position = UDim2.new(1, -50, 0.5, -10)
minimizeBtn.BackgroundColor3 = C.modeBtnBg
minimizeBtn.BorderSizePixel = 0
minimizeBtn.Text = "_"
minimizeBtn.TextColor3 = C.topSub
minimizeBtn.Font = Enum.Font.GothamBlack
minimizeBtn.TextSize = 14
minimizeBtn.ZIndex = 7
mkCorner(minimizeBtn, 3)
mkStroke(minimizeBtn, C.chipBorder, 1)
minimizeBtn.MouseEnter:Connect(function() TweenService:Create(minimizeBtn, TweenInfo.new(0.1), {TextColor3 = C.topTitle, BackgroundColor3 = Color3.fromRGB(20,50,20)}):Play() end)
minimizeBtn.MouseLeave:Connect(function() TweenService:Create(minimizeBtn, TweenInfo.new(0.1), {TextColor3 = C.topSub, BackgroundColor3 = C.modeBtnBg}):Play() end)
minimizeBtn.MouseButton1Click:Connect(function()
    State.guiMinimized = not State.guiMinimized
    if State.guiMinimized then
        TweenService:Create(mainOuter, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut), {
            Size = UDim2.new(0, WIN_W, 0, TITLE_H)
        }):Play()
        minimizeBtn.Text = "+"
    else
        TweenService:Create(mainOuter, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut), {
            Size = UDim2.new(0, WIN_W, 0, WIN_H)
        }):Play()
        minimizeBtn.Text = "_"
    end
end)

local titleDiv = Instance.new("Frame", mainOuter)
titleDiv.Size = UDim2.new(1, 0, 0, 1)
titleDiv.Position = UDim2.new(0, 0, 0, TITLE_H)
titleDiv.BackgroundColor3 = C.topDivider
titleDiv.BorderSizePixel = 0
titleDiv.ZIndex = 5

-- ============================================================
-- VERTICAL SIDEBAR + CONTENT AREA
-- ============================================================
local CONTENT_Y = TITLE_H + 1

-- Sidebar (left)
local tabBar = Instance.new("Frame", mainOuter)
tabBar.Name = "TabBar"
tabBar.Size = UDim2.new(0, SIDEBAR_W, 1, -CONTENT_Y)
tabBar.Position = UDim2.new(0, 0, 0, CONTENT_Y)
tabBar.BackgroundColor3 = C.tabBarBg
tabBar.BackgroundTransparency = 0
tabBar.BorderSizePixel = 0
tabBar.ZIndex = 5
tabBar.ClipsDescendants = true

-- Sidebar vertical divider
local sidebarDiv = Instance.new("Frame", mainOuter)
sidebarDiv.Size = UDim2.new(0, 1, 1, -CONTENT_Y)
sidebarDiv.Position = UDim2.new(0, SIDEBAR_W, 0, CONTENT_Y)
sidebarDiv.BackgroundColor3 = C.tabBarDiv
sidebarDiv.BorderSizePixel = 0
sidebarDiv.ZIndex = 5

local tabBarLL = Instance.new("UIListLayout", tabBar)
tabBarLL.FillDirection = Enum.FillDirection.Vertical
tabBarLL.SortOrder = Enum.SortOrder.LayoutOrder
tabBarLL.Padding = UDim.new(0, 2)

local tabBarPad = Instance.new("UIPadding", tabBar)
tabBarPad.PaddingTop = UDim.new(0, 8)
tabBarPad.PaddingLeft = UDim.new(0, 6)
tabBarPad.PaddingRight = UDim.new(0, 6)

-- Content area (right of sidebar)
local contentBg = Instance.new("Frame", mainOuter)
contentBg.Name = "ContentArea"
contentBg.Size = UDim2.new(1, -(SIDEBAR_W + 1), 1, -CONTENT_Y)
contentBg.Position = UDim2.new(0, SIDEBAR_W + 1, 0, CONTENT_Y)
contentBg.BackgroundColor3 = C.winBg
contentBg.BackgroundTransparency = 0
contentBg.BorderSizePixel = 0
contentBg.ClipsDescendants = true
contentBg.ZIndex = 2

-- ============================================================
-- LOGO WATERMARK (Radiant Hub logo in background like Aphex Hub)
-- ============================================================
local logoWatermark = Instance.new("ImageLabel", contentBg)
logoWatermark.Name = "LogoWatermark"
logoWatermark.Size = UDim2.new(1, 200, 1, 200)
logoWatermark.Position = UDim2.new(0, -100, 0, -100)
logoWatermark.BackgroundTransparency = 1
logoWatermark.Image = "rbxassetid://124121293012888"
logoWatermark.ImageTransparency = 0.80
logoWatermark.ScaleType = Enum.ScaleType.Fit
logoWatermark.ZIndex = 1

-- ============================================================
local TABS = {"Speed", "Combat", "Mechanics", "Movement", "Visual", "Settings"}
local currentTab = "Speed"
local tabBtns = {}; local tabPages = {}

for i, name in ipairs(TABS) do
-- Aphex Hub style: wrapper with left accent indicator
local btnWrap = Instance.new("Frame", tabBar)
btnWrap.Size = UDim2.new(1, 0, 0, 32)
btnWrap.BackgroundTransparency = 1
btnWrap.BorderSizePixel = 0
btnWrap.LayoutOrder = i

-- Left accent line (green when active, hidden when not)
local accentLine = Instance.new("Frame", btnWrap)
accentLine.Size = UDim2.new(0, 2, 1, 0)
accentLine.Position = UDim2.new(0, 0, 0, 0)
accentLine.BackgroundColor3 = (name == currentTab) and RED or C.tabBarBg
accentLine.BorderSizePixel = 0

local btn = Instance.new("TextButton", btnWrap)
btn.Size = UDim2.new(1, -4, 1, 0)
btn.Position = UDim2.new(0, 4, 0, 0)
btn.BackgroundColor3 = Color3.fromRGB(30, 40, 30)
btn.BackgroundTransparency = (name == currentTab) and 0 or 1
btn.BorderSizePixel = 0
btn.Text = name
btn.TextColor3 = (name == currentTab) and C.topTitle or C.tabIdle
btn.Font = Enum.Font.GothamBold
btn.TextSize = 11
btn.TextWrapped = true
btn.ZIndex = 6
mkCorner(btn, 4)

tabBtns[name] = {btn = btn, accent = accentLine, underline = nil}

btn.MouseEnter:Connect(function()
    if name ~= currentTab then
        TweenService:Create(btn, TweenInfo.new(0.1), {
            BackgroundTransparency = 0,
            BackgroundColor3 = Color3.fromRGB(26, 35, 26),
            TextColor3 = Color3.fromRGB(200, 230, 200)
        }):Play()
    end
end)
btn.MouseLeave:Connect(function()
    if name ~= currentTab then
        TweenService:Create(btn, TweenInfo.new(0.1), {
            BackgroundTransparency = 1,
            TextColor3 = C.tabIdle
        }):Play()
    end
end)
btn.MouseButton1Click:Connect(function()
    currentTab = name
    for _, n in ipairs(TABS) do
        local t = tabBtns[n]; local active = (n == name)
        TweenService:Create(t.btn, TweenInfo.new(0.14), {
            TextColor3 = active and C.topTitle or C.tabIdle,
            BackgroundColor3 = Color3.fromRGB(30, 40, 30),
            BackgroundTransparency = active and 0 or 1
        }):Play()
        TweenService:Create(t.accent, TweenInfo.new(0.14), {
            BackgroundColor3 = active and RED or C.tabBarBg
        }):Play()
        if tabPages[n] then tabPages[n].Visible = active end
    end
end)
end

-- ============================================================
-- ROW / PAGE BUILDERS
-- ============================================================
local currentPage = nil; local lo = 0
local function LO() lo = lo + 1; return lo end

local function makeGap(px)
local f = Instance.new("Frame", currentPage)
f.Size = UDim2.new(1, 0, 0, px or 6)
f.BackgroundTransparency = 1; f.BorderSizePixel = 0; f.LayoutOrder = LO()
end

local function makeSectionHeader(label)
local wrap = Instance.new("Frame", currentPage)
wrap.Size = UDim2.new(1, 0, 0, 28)
wrap.BackgroundTransparency = 1; wrap.BorderSizePixel = 0; wrap.LayoutOrder = LO()
local lbl = Instance.new("TextLabel", wrap)
lbl.Size = UDim2.new(1, -28, 1, 0)
lbl.Position = UDim2.new(0, 14, 0, 0)
lbl.BackgroundTransparency = 1
lbl.Text = label and label:upper() or ""
lbl.TextColor3 = C.sectionTxt
lbl.Font = Enum.Font.GothamBold
lbl.TextSize = 10
lbl.TextXAlignment = Enum.TextXAlignment.Left
end

local function makeInputRow(label, default, onChange)
local row = Instance.new("Frame", currentPage)
row.Size = UDim2.new(1, 0, 0, 44)
row.BackgroundColor3 = C.rowBg
row.BackgroundTransparency = 0
row.BorderSizePixel = 0
row.LayoutOrder = LO()

local div = Instance.new("Frame", row)
div.Size = UDim2.new(1, -28, 0, 1)
div.Position = UDim2.new(0, 14, 1, -1)
div.BackgroundColor3 = C.rowBorder; div.BorderSizePixel = 0

local lbl = Instance.new("TextLabel", row)
lbl.Size = UDim2.new(1, -100, 1, 0)
lbl.Position = UDim2.new(0, 14, 0, 0)
lbl.BackgroundTransparency = 1
lbl.Text = label
lbl.TextColor3 = C.rowLabel
lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 13
lbl.TextXAlignment = Enum.TextXAlignment.Left

local boxWrap = Instance.new("Frame", row)
boxWrap.Size = UDim2.new(0, 70, 0, 28)
boxWrap.Position = UDim2.new(1, -84, 0.5, -14)
boxWrap.BackgroundColor3 = C.inputBg; boxWrap.BorderSizePixel = 0
mkCorner(boxWrap, 5)
local bs = mkStroke(boxWrap, C.inputBorder, 1)

local box = Instance.new("TextBox", boxWrap)
box.Size = UDim2.new(1, -8, 1, 0); box.Position = UDim2.new(0, 4, 0, 0)
box.BackgroundTransparency = 1; box.Text = tostring(default)
box.TextColor3 = C.inputTxt; box.Font = Enum.Font.GothamBold
box.TextSize = 13; box.ClearTextOnFocus = false; box.ZIndex = 8
box.TextXAlignment = Enum.TextXAlignment.Center
box.Focused:Connect(function() TweenService:Create(bs, TweenInfo.new(0.15), {Color=C.inputFocus}):Play() end)
box.FocusLost:Connect(function()
    TweenService:Create(bs, TweenInfo.new(0.15), {Color=C.inputBorder}):Play()
    if onChange then local n = tonumber(box.Text); if n then onChange(n) else box.Text = tostring(default) end end
end)
return box, row
end

local function makeToggleRow(label, defaultOn, onToggle)
local row = Instance.new("Frame", currentPage)
row.Size = UDim2.new(1, 0, 0, 44)
row.BackgroundTransparency = 1; row.BorderSizePixel = 0; row.LayoutOrder = LO()

local div = Instance.new("Frame", row)
div.Size = UDim2.new(1, -28, 0, 1); div.Position = UDim2.new(0, 14, 1, -1)
div.BackgroundColor3 = C.rowBorder; div.BorderSizePixel = 0

local lbl = Instance.new("TextLabel", row)
lbl.Size = UDim2.new(1, -70, 1, 0); lbl.Position = UDim2.new(0, 14, 0, 0)
lbl.BackgroundTransparency = 1; lbl.Text = label
lbl.TextColor3 = C.rowLabel; lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 13
lbl.TextXAlignment = Enum.TextXAlignment.Left

local pillBg = Instance.new("Frame", row)
pillBg.Size = UDim2.new(0, 40, 0, 20); pillBg.Position = UDim2.new(1, -54, 0.5, -10)
pillBg.BackgroundColor3 = defaultOn and C.pillOn or C.pillOff
pillBg.BorderSizePixel = 0; pillBg.ZIndex = 7
mkCorner(pillBg, 10); mkStroke(pillBg, C.pillBorder, 1)

local dot = Instance.new("Frame", pillBg)
dot.Size = UDim2.new(0, 14, 0, 14)
dot.Position = defaultOn and UDim2.new(1, -17, 0.5, -7) or UDim2.new(0, 3, 0.5, -7)
dot.BackgroundColor3 = defaultOn and C.dotOn or C.dotOff
dot.BorderSizePixel = 0; dot.ZIndex = 8; mkCorner(dot, 7)

local isOn = defaultOn or false
local function setV(on, skipCallback)
    isOn = on
    TweenService:Create(pillBg, TweenInfo.new(0.18, Enum.EasingStyle.Quad), {BackgroundColor3 = on and C.pillOn or C.pillOff}):Play()
    TweenService:Create(dot, TweenInfo.new(0.18, Enum.EasingStyle.Back), {
        Position = on and UDim2.new(1, -17, 0.5, -7) or UDim2.new(0, 3, 0.5, -7),
        BackgroundColor3 = on and C.dotOn or C.dotOff
    }):Play()
    if not skipCallback and onToggle then pcall(onToggle, on) end
end
local function toggle() isOn = not isOn; setV(isOn); end

local clk = Instance.new("TextButton", row)
clk.Size = UDim2.new(1, -58, 1, 0); clk.BackgroundTransparency = 1
clk.Text = ""; clk.ZIndex = 5; clk.BorderSizePixel = 0
clk.MouseButton1Click:Connect(toggle)
local pClk = Instance.new("TextButton", pillBg)
pClk.Size = UDim2.new(1, 0, 1, 0); pClk.BackgroundTransparency = 1
pClk.Text = ""; pClk.ZIndex = 9; pClk.BorderSizePixel = 0
pClk.MouseButton1Click:Connect(toggle)
return setV
end

-- ============================================================
-- KEYBIND ROW
-- ============================================================
local getKeyDisplayName = WYNF_NO_VIRTUALIZE(function(kc)
local n = kc.Name
local gpNames = {
ButtonA="A",ButtonB="B",ButtonX="X",ButtonY="Y",
ButtonL1="LB",ButtonL2="LT",ButtonL3="LS",
ButtonR1="RB",ButtonR2="RT",ButtonR3="RS",
ButtonSelect="SEL",ButtonStart="STA",
DPadUp="D↑",DPadDown="D↓",DPadLeft="D←",DPadRight="D→",
Thumbstick1="LS",Thumbstick2="RS",
}
if gpNames[n] then return gpNames[n] end
return n:sub(1, 5)
end)

local function makeKeybindRow(label, currentKey, onChanged, keyName)
local row = Instance.new("Frame", currentPage)
row.Size = UDim2.new(1, 0, 0, 44)
row.BackgroundTransparency = 1; row.BorderSizePixel = 0; row.LayoutOrder = LO()

local div = Instance.new("Frame", row)
div.Size = UDim2.new(1, -28, 0, 1); div.Position = UDim2.new(0, 14, 1, -1)
div.BackgroundColor3 = C.rowBorder; div.BorderSizePixel = 0

local lbl = Instance.new("TextLabel", row)
lbl.Size = UDim2.new(1, -80, 1, 0); lbl.Position = UDim2.new(0, 14, 0, 0)
lbl.BackgroundTransparency = 1; lbl.Text = label
lbl.TextColor3 = C.rowLabel; lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 13
lbl.TextXAlignment = Enum.TextXAlignment.Left

local kbtn = Instance.new("TextButton", row)
kbtn.Size = UDim2.new(0, 52, 0, 26); kbtn.Position = UDim2.new(1, -64, 0.5, -13)
kbtn.BackgroundColor3 = C.chipBg; kbtn.BorderSizePixel = 0
kbtn.Text = getKeyDisplayName(currentKey); kbtn.TextColor3 = C.chipTxt
kbtn.Font = Enum.Font.GothamBold; kbtn.TextSize = 11; kbtn.ZIndex = 8
mkCorner(kbtn, 5); local ks = mkStroke(kbtn, C.chipBorder, 1)

local listening = false; local lconnKeyboard = nil; local lconnGamepad = nil
local function stopL(key)
    listening = false
    if lconnKeyboard then lconnKeyboard:Disconnect(); lconnKeyboard = nil end
    if lconnGamepad  then lconnGamepad:Disconnect();  lconnGamepad = nil  end
    TweenService:Create(ks, TweenInfo.new(0.12), {Color=C.chipBorder}):Play()
    kbtn.TextColor3 = C.chipTxt
    if key then
        kbtn.Text = getKeyDisplayName(key)
        if onChanged then onChanged(key) end
        task.spawn(function() if saveConfig then pcall(saveConfig) end end)
    end
end
kbtn.MouseButton1Click:Connect(function()
    if listening then stopL(nil); return end
    listening = true; kbtn.Text = "···"; kbtn.TextColor3 = C.inputTxt
    TweenService:Create(ks, TweenInfo.new(0.12), {Color=C.inputFocus}):Play()
    lconnKeyboard = UIS.InputBegan:Connect(function(inp)
        if not listening then return end
        if inp.UserInputType ~= Enum.UserInputType.Keyboard then return end
        if inp.KeyCode == Enum.KeyCode.Escape then stopL(nil); return end
        stopL(inp.KeyCode)
    end)
    lconnGamepad = UIS.InputBegan:Connect(function(inp)
        if not listening then return end
        if inp.UserInputType ~= Enum.UserInputType.Gamepad1
        and inp.UserInputType ~= Enum.UserInputType.Gamepad2
        and inp.UserInputType ~= Enum.UserInputType.Gamepad3
        and inp.UserInputType ~= Enum.UserInputType.Gamepad4 then return end
        local kc = inp.KeyCode; if kc == Enum.KeyCode.Unknown then return end
        stopL(kc)
    end)
end)
if keyName then keybindBtnRefs[keyName] = kbtn end
return kbtn
end

-- ============================================================
-- BUILD PAGES
-- ============================================================
local function buildPage(tabName, buildFn)
local page = Instance.new("ScrollingFrame", contentBg)
page.Name = tabName; page.Visible = (tabName == "Speed")
page.Size = UDim2.new(1, 0, 1, 0); page.Position = UDim2.new(0, 0, 0, 0)
page.BackgroundTransparency = 1; page.BorderSizePixel = 0
page.ScrollBarThickness = 3
page.ScrollBarImageColor3 = C.tabUnderline
page.ScrollBarImageTransparency = 0.4
page.AutomaticCanvasSize = Enum.AutomaticSize.Y; page.CanvasSize = UDim2.new(0, 0, 0, 0)
local ll = Instance.new("UIListLayout", page)
ll.SortOrder = Enum.SortOrder.LayoutOrder; ll.Padding = UDim.new(0, 0)
tabPages[tabName] = page; currentPage = page; lo = 0
buildFn()
currentPage = nil
end

-- ============================================================
-- MECHANICS PAGE (NOW FIRST)
-- ============================================================
buildPage("Mechanics", function()
makeGap(2)
makeSectionHeader("Stealing")
makeGap(2)
setInstaGrab = makeToggleRow("Insta Grab", false, function(on)
Steal.AutoStealEnabled = on
if on then if not pcall(startAutoSteal) then Steal.AutoStealEnabled = false; setInstaGrab(false) end
else stopAutoSteal() end
end)
stealRadBox = makeInputRow("Steal Radius", Steal.StealRadius, function(n)
if n >= 5 and n <= 300 then Steal.StealRadius = math.floor(n); Steal.cachedPrompts = {}; Steal.promptCacheTime = 0
if radTB and not radTB:IsFocused() then radTB.Text = tostring(Steal.StealRadius) end end
end)
stealDurBox = makeInputRow("Steal Duration", Steal.StealDuration, function(n)
if n >= 0.05 and n <= 10 then Steal.StealDuration = n end
end)
makeGap(8)
makeSectionHeader("Movement Options")
makeGap(2)
setInfJump = makeToggleRow("Infinite Jump", false, function(on) State.infJumpEnabled = on end)
setAntiRag = makeToggleRow("Anti Ragdoll", false, function(on)
State.antiRagdollEnabled = on; if on then startAntiRagdoll() else stopAntiRagdoll() end
end)
makeGap(8)
makeSectionHeader("Auto TP Down")
makeGap(2)
setAutoTpDown = makeToggleRow("Auto TP Down", false, function(on)
    State.autoTpDownEnabled = on
end)
autoTpDownHeightBox = makeInputRow("Y Trigger", State.autoTpDownHeight, function(n)
    if n >= 1 and n <= 500 then State.autoTpDownHeight = n end
end)
makeGap(8)
end)

-- ============================================================
-- SPEED PAGE
-- ============================================================
buildPage("Speed", function()
makeGap(2)
makeSectionHeader("Speed Settings")
makeGap(2)

normalBox = makeInputRow("Normal Speed", State.normalSpeed, function(n)
    if n > 0 and n <= 500 then State.normalSpeed = n end
end)
carryBox = makeInputRow("Carry Speed", State.carrySpeed, function(n)
    if n > 0 and n <= 500 then State.carrySpeed = n end
end)
laggerBox = makeInputRow("Lagger Speed", State.laggerSpeed, function(n)
    if n > 0 and n <= 500 then State.laggerSpeed = n end
end)

makeGap(6)

local modeRow = Instance.new("Frame", currentPage)
modeRow.Size = UDim2.new(1, 0, 0, 48)
modeRow.BackgroundTransparency = 1; modeRow.BorderSizePixel = 0; modeRow.LayoutOrder = LO()

local modeWrap = Instance.new("Frame", modeRow)
modeWrap.Size = UDim2.new(1, -28, 0, 34)
modeWrap.Position = UDim2.new(0, 14, 0, 7)
modeWrap.BackgroundColor3 = C.modeBtnBg; modeWrap.BorderSizePixel = 0
mkCorner(modeWrap, 7); mkStroke(modeWrap, C.modeBtnBrd, 1)

local modeLL = Instance.new("UIListLayout", modeWrap)
modeLL.FillDirection = Enum.FillDirection.Horizontal
modeLL.SortOrder = Enum.SortOrder.LayoutOrder; modeLL.Padding = UDim.new(0, 0)

local modeStatusRow = Instance.new("Frame", currentPage)
modeStatusRow.Size = UDim2.new(1, 0, 0, 24)
modeStatusRow.BackgroundTransparency = 1; modeStatusRow.BorderSizePixel = 0; modeStatusRow.LayoutOrder = LO()
local modeStatusLbl = Instance.new("TextLabel", modeStatusRow)
modeStatusLbl.Size = UDim2.new(1, -28, 1, 0); modeStatusLbl.Position = UDim2.new(0, 14, 0, 0)
modeStatusLbl.BackgroundTransparency = 1; modeStatusLbl.Text = "Mode: Normal"
modeStatusLbl.TextColor3 = C.rowSub; modeStatusLbl.Font = Enum.Font.Gotham
modeStatusLbl.TextSize = 11; modeStatusLbl.TextXAlignment = Enum.TextXAlignment.Left

local modeNames = {"Normal", "Carry", "Lagger"}
local modeBtns = {}
local function setModeActive(active)
    for _, m in ipairs(modeNames) do
        local b = modeBtns[m]; if not b then continue end
        local isActive = (m == active)
        TweenService:Create(b, TweenInfo.new(0.15), {
            BackgroundColor3 = isActive and C.modeBtnActBg or Color3.fromRGB(25,25,25),
            BackgroundTransparency = isActive and 0 or 1,
            TextColor3 = isActive and C.modeBtnActTx or C.modeBtnTxt,
        }):Play()
    end
    modeStatusLbl.Text = "Mode: " .. active
    if active == "Normal" then
        State.speedToggled = false; State.laggerEnabled = false
        if stackBtnRefs.carrySpeed then stackBtnRefs.carrySpeed.setOn(false) end
        if stackBtnRefs.lagger then stackBtnRefs.lagger.setOn(false) end
    elseif active == "Carry" then
        State.speedToggled = true; State.laggerEnabled = false
        if stackBtnRefs.carrySpeed then stackBtnRefs.carrySpeed.setOn(true) end
        if stackBtnRefs.lagger then stackBtnRefs.lagger.setOn(false) end
    elseif active == "Lagger" then
        State.speedToggled = false; State.laggerEnabled = true
        if stackBtnRefs.carrySpeed then stackBtnRefs.carrySpeed.setOn(false) end
        if stackBtnRefs.lagger then stackBtnRefs.lagger.setOn(true) end
    end
end

for i, mname in ipairs(modeNames) do
    local b = Instance.new("TextButton", modeWrap)
    b.Size = UDim2.new(1/3, 0, 1, 0)
    b.BackgroundColor3 = (i == 1) and C.modeBtnActBg or Color3.fromRGB(255,255,255)
    b.BackgroundTransparency = (i == 1) and 0 or 1
    b.BorderSizePixel = 0; b.Text = mname
    b.TextColor3 = (i == 1) and C.modeBtnActTx or C.modeBtnTxt
    b.Font = Enum.Font.GothamBold; b.TextSize = 12; b.ZIndex = 8
    b.LayoutOrder = i; mkCorner(b, 5)
    b.MouseButton1Click:Connect(function() setModeActive(mname) end)
    modeBtns[mname] = b
end

makeGap(6)
makeSectionHeader("Keybinds")
makeGap(2)
makeKeybindRow("Speed Key", Keys.speed, function(k) Keys.speed = k end, "speed")
makeKeybindRow("Lagger Key", Keys.lagger, function(k) Keys.lagger = k end, "lagger")
end)

-- ============================================================
-- BAT AIMBOT PAGE
-- ============================================================
buildPage("Combat", function()
makeGap(2)
makeSectionHeader("Combat Systems")
makeGap(2)
setAutoSwing = makeToggleRow("Auto Swing", false, function(on) State.autoSwingEnabled = on end)
setBatCounter = makeToggleRow("Bat Counter", false, function(on)
State.batCounterEnabled = on
if on then startBatCounter() else stopBatCounter() end
end)
setMedusaCounter = makeToggleRow("Medusa Counter", false, function(on)
State.medusaCounterEnabled = on
if on then setupMedusaCounter(LP.Character) else stopMedusaCounter() end
end)
makeGap(8)
makeSectionHeader("Keybinds")
makeGap(2)
makeKeybindRow("Aimbot Key", Keys.aimbot, function(k) Keys.aimbot = k end, "aimbot")
end)

-- ============================================================
-- MOVEMENT PAGE
-- ============================================================
buildPage("Movement", function()
makeGap(2)
makeSectionHeader("Auto Movement")
makeGap(2)
makeKeybindRow("Auto Left", Keys.autoLeft, function(k) Keys.autoLeft = k end, "autoLeft")
makeKeybindRow("Auto Right", Keys.autoRight, function(k) Keys.autoRight = k end, "autoRight")
makeGap(8)

makeSectionHeader("Play Mode")
makeGap(2)
-- Mode: Full / Semi Auto Play toggle
local modeBtnRow = Instance.new("Frame", currentPage)
modeBtnRow.Size = UDim2.new(1, 0, 0, 44)
modeBtnRow.BackgroundTransparency = 1; modeBtnRow.BorderSizePixel = 0
modeBtnRow.LayoutOrder = LO()
local modeBtnDiv = Instance.new("Frame", modeBtnRow)
modeBtnDiv.Size = UDim2.new(1, -28, 0, 1); modeBtnDiv.Position = UDim2.new(0, 14, 1, -1)
modeBtnDiv.BackgroundColor3 = C.rowBorder; modeBtnDiv.BorderSizePixel = 0
modeBtn = Instance.new("TextButton", modeBtnRow)
modeBtn.Name = "RadiantModeBtn"
modeBtn.Size = UDim2.new(1, -28, 0, 28)
modeBtn.Position = UDim2.new(0, 14, 0.5, -14)
modeBtn.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
modeBtn.BorderSizePixel = 0
modeBtn.Text = "Mode: Full Auto Play"
modeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
modeBtn.Font = Enum.Font.GothamBold
modeBtn.TextSize = 13
modeBtn.AutoButtonColor = false
modeBtn.ZIndex = 6
mkCorner(modeBtn, 8)
mkStroke(modeBtn, Color3.fromRGB(60, 60, 60), 1)
modeBtn.MouseButton1Click:Connect(function()
	if modeToggleState == "full" then
		modeToggleState = "semi"
		modeBtn.Text = "Mode: Semi Auto Play"
	else
		modeToggleState = "full"
		modeBtn.Text = "Mode: Full Auto Play"
		if State.autoLeftEnabled then State.autoLeftEnabled = false; stopAutoLeft() end
		if State.autoRightEnabled then State.autoRightEnabled = false; stopAutoRight() end
	end
	local char = LP.Character
	if char then
		local head = char:FindFirstChild("Head")
		local bb = head and head:FindFirstChild("RadiantHubBB")
		local modeLbl = bb and bb:FindFirstChild("ModeBillLbl")
		if modeLbl then modeLbl.Text = modeToggleState == "semi" and "Mode: Semi" or "Mode: Full" end
	end
end)
makeGap(8)

makeSectionHeader("Duel Countdown")
makeGap(2)

local duelDir = "left"
local dirRow = Instance.new("Frame", currentPage)
dirRow.Size = UDim2.new(1, 0, 0, 48)
dirRow.BackgroundTransparency = 1; dirRow.BorderSizePixel = 0; dirRow.LayoutOrder = LO()

local dirLbl = Instance.new("TextLabel", dirRow)
dirLbl.Size = UDim2.new(0.5, -14, 1, 0); dirLbl.Position = UDim2.new(0, 14, 0, 0)
dirLbl.BackgroundTransparency = 1; dirLbl.Text = "Direction"
dirLbl.TextColor3 = C.rowLabel; dirLbl.Font = Enum.Font.GothamBold; dirLbl.TextSize = 13
dirLbl.TextXAlignment = Enum.TextXAlignment.Left

local dirWrap = Instance.new("Frame", dirRow)
dirWrap.Size = UDim2.new(0, 110, 0, 28); dirWrap.Position = UDim2.new(1, -124, 0.5, -14)
dirWrap.BackgroundColor3 = C.modeBtnBg; dirWrap.BorderSizePixel = 0
mkCorner(dirWrap, 6); mkStroke(dirWrap, C.modeBtnBrd, 1)
local dirLL = Instance.new("UIListLayout", dirWrap)
dirLL.FillDirection = Enum.FillDirection.Horizontal
dirLL.SortOrder = Enum.SortOrder.LayoutOrder; dirLL.Padding = UDim.new(0, 0)

local dirDivRow = Instance.new("Frame", dirRow)
dirDivRow.Size = UDim2.new(1, -28, 0, 1); dirDivRow.Position = UDim2.new(0, 14, 1, -1)
dirDivRow.BackgroundColor3 = C.rowBorder; dirDivRow.BorderSizePixel = 0

local dirBtns = {}
for i, dname in ipairs({"Left", "Right"}) do
    local db = Instance.new("TextButton", dirWrap)
    db.Size = UDim2.new(0.5, 0, 1, 0)
    db.BackgroundColor3 = (i==1) and C.modeBtnActBg or Color3.fromRGB(255,255,255)
    db.BackgroundTransparency = (i==1) and 0 or 1
    db.BorderSizePixel = 0
    db.Text = dname; db.TextColor3 = (i==1) and C.modeBtnActTx or C.modeBtnTxt
    db.Font = Enum.Font.GothamBold; db.TextSize = 11; db.ZIndex = 6
    db.LayoutOrder = i
    dirBtns[dname] = db
end
local function setDirActive(active)
    for _, dname in ipairs({"Left","Right"}) do
        local b = dirBtns[dname]; if not b then continue end
        local isA = (dname == active)
        TweenService:Create(b, TweenInfo.new(0.15), {
            BackgroundColor3 = isA and C.modeBtnActBg or Color3.fromRGB(255,255,255),
            BackgroundTransparency = isA and 0 or 1,
            TextColor3 = isA and C.modeBtnActTx or C.modeBtnTxt,
        }):Play()
    end
    duelDir = active:lower()
    if State.duelCountdownEnabled then
        stopDuelCountdownWatcher()
        startDuelCountdownWatcher(duelDir)
    end
end
dirBtns["Left"].MouseButton1Click:Connect(function() setDirActive("Left") end)
dirBtns["Right"].MouseButton1Click:Connect(function() setDirActive("Right") end)

makeToggleRow("Auto on Countdown End", false, function(on)
    State.duelCountdownEnabled = on
    if on then
        startDuelCountdownWatcher(duelDir)
    else
        stopDuelCountdownWatcher()
    end
end)

local infoRow = Instance.new("Frame", currentPage)
infoRow.Size = UDim2.new(1, 0, 0, 28)
infoRow.BackgroundTransparency = 1; infoRow.BorderSizePixel = 0; infoRow.LayoutOrder = LO()
local infoLbl = Instance.new("TextLabel", infoRow)
infoLbl.Size = UDim2.new(1, -28, 1, 0); infoLbl.Position = UDim2.new(0, 14, 0, 0)
infoLbl.BackgroundTransparency = 1
infoLbl.Text = "Fires auto move when duel countdown ends"
infoLbl.TextColor3 = C.rowSub; infoLbl.Font = Enum.Font.Gotham; infoLbl.TextSize = 10
infoLbl.TextXAlignment = Enum.TextXAlignment.Left

makeGap(8)
makeSectionHeader("Other Keys")
makeGap(2)
makeKeybindRow("Drop Key", Keys.drop, function(k) Keys.drop = k end, "drop")
makeKeybindRow("TP Down Key", Keys.tpDown, function(k) Keys.tpDown = k end, "tpDown")
end)


-- ============================================================
-- VISUAL PAGE
-- ============================================================
buildPage("Visual", function()
makeGap(2)
makeSectionHeader("Animations")
makeGap(2)
setNoAnim = makeToggleRow("No Animation", false, function(on)
State.noAnimEnabled = on
if on then pcall(function() startNoAnim(LP.Character) end) else pcall(stopNoAnim) end
end)
setTryhardAnim = makeToggleRow("Tryhard Anim", false, function(on)
State.animEnabled = on
if on then startAnimToggle() else stopAnimToggle() end
end)
makeGap(8)
makeSectionHeader("Camera")
makeGap(2)
setStretchRez = makeToggleRow("Stretch Rez", false, function(on)
    State.stretchRezEnabled = on
    local cam = workspace.CurrentCamera
    if cam then cam.FieldOfView = on and State.stretchRezFov or 70 end
end)
-- FOV slider row
do
    local FOV_MIN, FOV_MAX = 0, 120
    local fovRow = Instance.new("Frame", currentPage)
    fovRow.Size = UDim2.new(1, 0, 0, 62)
    fovRow.BackgroundColor3 = C.rowBg
    fovRow.BackgroundTransparency = 0
    fovRow.BorderSizePixel = 0
    fovRow.LayoutOrder = LO()
    mkCorner(fovRow, 6)
    mkStroke(fovRow, C.rowBorder, 1)

    -- label + value display
    local fovLbl = Instance.new("TextLabel", fovRow)
    fovLbl.Size = UDim2.new(1, -90, 0, 22)
    fovLbl.Position = UDim2.new(0, 14, 0, 6)
    fovLbl.BackgroundTransparency = 1
    fovLbl.Text = "FOV (0–120)"
    fovLbl.TextColor3 = C.rowLabel
    fovLbl.Font = Enum.Font.GothamBold
    fovLbl.TextSize = 13
    fovLbl.TextXAlignment = Enum.TextXAlignment.Left

    local fovBoxWrap = Instance.new("Frame", fovRow)
    fovBoxWrap.Size = UDim2.new(0, 52, 0, 22)
    fovBoxWrap.Position = UDim2.new(1, -66, 0, 6)
    fovBoxWrap.BackgroundColor3 = C.inputBg
    fovBoxWrap.BorderSizePixel = 0
    mkCorner(fovBoxWrap, 5)
    local fovBoxStroke = mkStroke(fovBoxWrap, C.inputBorder, 1)

    stretchRezFovBox = Instance.new("TextBox", fovBoxWrap)
    stretchRezFovBox.Size = UDim2.new(1, -6, 1, 0)
    stretchRezFovBox.Position = UDim2.new(0, 3, 0, 0)
    stretchRezFovBox.BackgroundTransparency = 1
    stretchRezFovBox.Text = tostring(State.stretchRezFov)
    stretchRezFovBox.TextColor3 = C.inputTxt
    stretchRezFovBox.Font = Enum.Font.GothamBold
    stretchRezFovBox.TextSize = 13
    stretchRezFovBox.ClearTextOnFocus = false
    stretchRezFovBox.ZIndex = 8
    stretchRezFovBox.TextXAlignment = Enum.TextXAlignment.Center

    -- slider track
    local trackBg = Instance.new("Frame", fovRow)
    trackBg.Size = UDim2.new(1, -28, 0, 8)
    trackBg.Position = UDim2.new(0, 14, 0, 38)
    trackBg.BackgroundColor3 = C.inputBg
    trackBg.BorderSizePixel = 0
    mkCorner(trackBg, 4)
    mkStroke(trackBg, C.rowBorder, 1)

    local trackFill = Instance.new("Frame", trackBg)
    trackFill.Size = UDim2.new((State.stretchRezFov - FOV_MIN) / (FOV_MAX - FOV_MIN), 0, 1, 0)
    trackFill.BackgroundColor3 = C.pillOn
    trackFill.BorderSizePixel = 0
    mkCorner(trackFill, 4)

    local thumb = Instance.new("Frame", trackBg)
    thumb.Size = UDim2.new(0, 16, 0, 16)
    thumb.Position = UDim2.new((State.stretchRezFov - FOV_MIN) / (FOV_MAX - FOV_MIN), -8, 0.5, -8)
    thumb.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    thumb.BorderSizePixel = 0
    thumb.ZIndex = 4
    mkCorner(thumb, 8)

    local function applyFov(v)
        v = math.clamp(math.floor(v), FOV_MIN, FOV_MAX)
        State.stretchRezFov = v
        stretchRezFovBox.Text = tostring(v)
        local pct = (v - FOV_MIN) / (FOV_MAX - FOV_MIN)
        trackFill.Size = UDim2.new(pct, 0, 1, 0)
        thumb.Position = UDim2.new(pct, -8, 0.5, -8)
        if State.stretchRezEnabled then
            local cam = workspace.CurrentCamera
            if cam then cam.FieldOfView = v end
        end
    end

    -- textbox input
    stretchRezFovBox.Focused:Connect(function() TweenService:Create(fovBoxStroke, TweenInfo.new(0.15), {Color=C.inputFocus}):Play() end)
    stretchRezFovBox.FocusLost:Connect(function()
        TweenService:Create(fovBoxStroke, TweenInfo.new(0.15), {Color=C.inputBorder}):Play()
        local n = tonumber(stretchRezFovBox.Text)
        if n then applyFov(n) else stretchRezFovBox.Text = tostring(State.stretchRezFov) end
    end)

    -- slider drag
    local sliding = false
    local function scrubFromInput(inputPos)
        local abs = trackBg.AbsolutePosition
        local sz  = trackBg.AbsoluteSize
        local pct = math.clamp((inputPos.X - abs.X) / sz.X, 0, 1)
        applyFov(FOV_MIN + pct * (FOV_MAX - FOV_MIN))
    end
    trackBg.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 or inp.UserInputType == Enum.UserInputType.Touch then
            sliding = true; scrubFromInput(inp.Position)
        end
    end)
    UIS.InputChanged:Connect(function(inp)
        if not sliding then return end
        if inp.UserInputType == Enum.UserInputType.MouseMovement or inp.UserInputType == Enum.UserInputType.Touch then
            scrubFromInput(inp.Position)
        end
    end)
    UIS.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 or inp.UserInputType == Enum.UserInputType.Touch then
            sliding = false
        end
    end)
end
makeGap(8)
makeSectionHeader("Lighting")
makeGap(2)
setNightTime = makeToggleRow("Night Time", false, function(on)
    State.nightTimeEnabled = on
    local Lighting = game:GetService("Lighting")
    if on then
        Lighting.ClockTime = 0
        Lighting.Brightness = 0.2
        Lighting.Ambient = Color3.fromRGB(20, 20, 35)
    else
        Lighting.ClockTime = 14
        Lighting.Brightness = 2
        Lighting.Ambient = Color3.fromRGB(128, 128, 128)
    end
end)
setPurpleSky = makeToggleRow("Red Sky", false, function(on)
    State.purpleSkyEnabled = on
    local Lighting = game:GetService("Lighting")
    if on then
        Lighting.Ambient = Color3.fromRGB(90, 10, 10)
        Lighting.FogColor = Color3.fromRGB(120, 20, 20)
        Lighting.FogEnd = 800
        local cc = Lighting:FindFirstChildOfClass("ColorCorrectionEffect") or Instance.new("ColorCorrectionEffect")
        cc.TintColor = Color3.fromRGB(255, 80, 80)
        cc.Saturation = 0.8
        cc.Contrast = 0.2
        cc.Parent = Lighting
    else
        Lighting.Ambient = Color3.fromRGB(128, 128, 128)
        Lighting.FogColor = Color3.fromRGB(191, 191, 191)
        Lighting.FogEnd = 100000
        local cc = Lighting:FindFirstChildOfClass("ColorCorrectionEffect")
        if cc then cc:Destroy() end
    end
end)
makeGap(8)
makeSectionHeader("Performance")
makeGap(2)
setFpsBooster = makeToggleRow("FPS Booster", false, function(on)
    State.fpsBoosterEnabled = on
    local Lighting = game:GetService("Lighting")
    if on then
        settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
        Lighting.GlobalShadows = false
        Lighting.Brightness = 3
        Lighting.FogEnd = 9000000000
        Lighting.Technology = Enum.Technology.Legacy
        Lighting.EnvironmentDiffuseScale = 0
        Lighting.EnvironmentSpecularScale = 0
        for _, plr in ipairs(Players:GetPlayers()) do
            local char = plr.Character
            if char then
                for _, acc in ipairs(char:GetChildren()) do
                    if acc:IsA("Accessory") then acc:Destroy() end
                end
            end
        end
    else
        settings().Rendering.QualityLevel = Enum.QualityLevel.Automatic
        Lighting.GlobalShadows = true
        Lighting.Brightness = 2
        Lighting.FogEnd = 100000
        Lighting.Technology = Enum.Technology.ShadowMap
        Lighting.EnvironmentDiffuseScale = 1
        Lighting.EnvironmentSpecularScale = 1
    end
end)
makeGap(8)
makeSectionHeader("Players")
makeGap(2)
-- ============================================================
-- BOX ESP (SelectionBox — scales correctly at any distance)
-- ============================================================
local _espFolder = Instance.new("Folder")
_espFolder.Name = "RadiantESPFolder"
_espFolder.Parent = workspace

local function addBoxESP(char, plr)
    if not char then return end
    if _espFolder:FindFirstChild("ESP_"..plr.Name) then return end

    -- SelectionBox wraps the whole character model perfectly at any distance
    local sel = Instance.new("SelectionBox", _espFolder)
    sel.Name = "ESP_"..plr.Name
    sel.Adornee = char
    sel.Color3 = Color3.fromRGB(0, 170, 255)
    sel.LineThickness = 0.04
    sel.SurfaceTransparency = 0.6
    sel.SurfaceColor3 = Color3.fromRGB(0, 120, 255)

    -- Name label above head via BillboardGui (small, just the name)
    local head2 = char:FindFirstChild("Head")
    if head2 then
        local bb = Instance.new("BillboardGui", head2)
        bb.Name = "RadiantESPName"
        bb.Size = UDim2.new(0, 100, 0, 20)
        bb.StudsOffset = Vector3.new(0, 2.5, 0)
        bb.AlwaysOnTop = true
        bb.ResetOnSpawn = false
        local nameLbl = Instance.new("TextLabel", bb)
        nameLbl.Size = UDim2.new(1, 0, 1, 0)
        nameLbl.BackgroundTransparency = 1
        nameLbl.Text = plr.Name
        nameLbl.TextColor3 = Color3.fromRGB(0, 200, 255)
        nameLbl.Font = Enum.Font.GothamBold
        nameLbl.TextSize = 13
        nameLbl.TextStrokeTransparency = 0.3
        nameLbl.TextStrokeColor3 = Color3.fromRGB(0,0,0)
    end
end

local function removeBoxESP(plr)
    local sel = _espFolder:FindFirstChild("ESP_"..plr.Name)
    if sel then sel:Destroy() end
    if plr.Character then
        local head2 = plr.Character:FindFirstChild("Head")
        if head2 then
            local bb = head2:FindFirstChild("RadiantESPName")
            if bb then bb:Destroy() end
        end
    end
end

local function espSetupPlayer(plr)
    if plr == LP then return end
    -- Remove any stale box first, then re-add for current character
    removeBoxESP(plr)
    if plr.Character then
        task.wait(0.1)
        addBoxESP(plr.Character, plr)
    end
    -- Re-add box every time this player respawns
    local conn = plr.CharacterAdded:Connect(function(char)
        if not State.espEnabled then return end
        removeBoxESP(plr)
        task.wait(0.2)
        addBoxESP(char, plr)
    end)
    table.insert(_espConns, conn)
end

setPlayerESP = makeToggleRow("Player ESP", false, function(on)
    State.espEnabled = on
    if on then
        -- Setup all current players
        for _, plr in ipairs(Players:GetPlayers()) do
            espSetupPlayer(plr)
        end
        -- Setup any players that join after enabling
        _espConns.charAdded = Players.PlayerAdded:Connect(function(plr)
            if not State.espEnabled then return end
            espSetupPlayer(plr)
        end)
        -- Also handle players leaving — clean up their box
        _espConns.playerRemoving = Players.PlayerRemoving:Connect(function(plr)
            removeBoxESP(plr)
        end)
    else
        for _, plr in ipairs(Players:GetPlayers()) do
            removeBoxESP(plr)
        end
        if _espConns.charAdded then _espConns.charAdded:Disconnect(); _espConns.charAdded = nil end
        if _espConns.playerRemoving then _espConns.playerRemoving:Disconnect(); _espConns.playerRemoving = nil end
        for i, conn in ipairs(_espConns) do
            if typeof(conn) == "RBXScriptConnection" then conn:Disconnect() end
            _espConns[i] = nil
        end
    end
end)
end)
-- ============================================================
local function applyStackButtonsVisible(visible)
State.stackButtonsHidden = not visible
for _, wrapper in pairs(stackWrappers) do wrapper.Visible = visible end
end

local function applyPreset(data)
if data.normalSpeed then State.normalSpeed=data.normalSpeed; if normalBox then normalBox.Text=tostring(data.normalSpeed) end end
if data.carrySpeed then State.carrySpeed=data.carrySpeed; if carryBox then carryBox.Text=tostring(data.carrySpeed) end end
if data.laggerSpeed then State.laggerSpeed=data.laggerSpeed; if laggerBox then laggerBox.Text=tostring(data.laggerSpeed) end end
if data.stealRadius then
Steal.StealRadius=data.stealRadius; Steal.cachedPrompts={}; Steal.promptCacheTime=0
if stealRadBox and not stealRadBox:IsFocused() then stealRadBox.Text=tostring(data.stealRadius) end
if radTB and not radTB:IsFocused() then radTB.Text=tostring(data.stealRadius) end
end
if data.stealDuration then Steal.StealDuration=data.stealDuration end
if data.infJump~=nil and setInfJump then State.infJumpEnabled=data.infJump; setInfJump(data.infJump) end
if data.antiRagdoll~=nil and setAntiRag then State.antiRagdollEnabled=data.antiRagdoll; setAntiRag(data.antiRagdoll); if data.antiRagdoll then startAntiRagdoll() else stopAntiRagdoll() end end
if data.medusaCounter~=nil and setMedusaCounter then State.medusaCounterEnabled=data.medusaCounter; setMedusaCounter(data.medusaCounter); if data.medusaCounter then setupMedusaCounter(LP.Character) else stopMedusaCounter() end end
if data.batCounter~=nil and setBatCounter then State.batCounterEnabled=data.batCounter; setBatCounter(data.batCounter); if data.batCounter then startBatCounter() else stopBatCounter() end end
if data.autoSteal~=nil and setInstaGrab then
Steal.AutoStealEnabled=data.autoSteal; setInstaGrab(data.autoSteal)
if data.autoSteal then pcall(startAutoSteal) else stopAutoSteal() end
end
end

buildPage("Settings", function()
makeGap(2)
makeSectionHeader("Interface")
makeGap(2)
makeKeybindRow("Hide GUI", Keys.guiHide, function(k) Keys.guiHide = k end, "guiHide")
uiScaleBox = makeInputRow("UI Scale", 1.0, function(n)
if n >= 0.5 and n <= 2.0 then if uiScaleObj then uiScaleObj.Scale = n end end
end)
setLockUI = makeToggleRow("Lock UI", State.uiLocked, function(on)
State.uiLocked = on
-- Only locks button positions (drag is blocked in makeStackDraggable via State.uiLocked)
-- Buttons still work and can be pressed normally
end)

makeGap(8)

local rWrap = Instance.new("Frame", currentPage)
rWrap.Size = UDim2.new(1, 0, 0, 46); rWrap.BackgroundTransparency = 1
rWrap.BorderSizePixel = 0; rWrap.LayoutOrder = LO()
local resetBtn = Instance.new("TextButton", rWrap)
resetBtn.Size = UDim2.new(1, -28, 0, 32); resetBtn.Position = UDim2.new(0, 14, 0, 7)
resetBtn.BackgroundColor3 = C.btnBg; resetBtn.BorderSizePixel = 0
resetBtn.Text = "↺  Reset Button Positions"; resetBtn.TextColor3 = C.btnTxt
resetBtn.Font = Enum.Font.GothamBold; resetBtn.TextSize = 12; resetBtn.ZIndex = 5
mkCorner(resetBtn, 6); mkStroke(resetBtn, C.btnBorder, 1)
resetBtn.MouseEnter:Connect(function() TweenService:Create(resetBtn,TweenInfo.new(0.1),{BackgroundColor3=C.btnHov}):Play() end)
resetBtn.MouseLeave:Connect(function() TweenService:Create(resetBtn,TweenInfo.new(0.1),{BackgroundColor3=C.btnBg}):Play() end)
resetBtn.MouseButton1Click:Connect(function()
    for i, def in ipairs(stackDefs) do
        local wrapper = stackWrappers[def.key]
        if wrapper then TweenService:Create(wrapper,TweenInfo.new(0.35,Enum.EasingStyle.Back,Enum.EasingDirection.Out),{Position=getDefaultStackPos(i)}):Play() end
    end
    resetBtn.Text = "✓  Positions Reset!"
    task.delay(1.8, function() if resetBtn and resetBtn.Parent then resetBtn.Text = "↺  Reset Button Positions" end end)
end)

makeGap(8)
makeSectionHeader("Config")
makeGap(4)

local scWrap = Instance.new("Frame", currentPage)
scWrap.Size = UDim2.new(1, 0, 0, 46); scWrap.BackgroundTransparency = 1
scWrap.BorderSizePixel = 0; scWrap.LayoutOrder = LO()
local saveConfigBtn = Instance.new("TextButton", scWrap)
saveConfigBtn.Size = UDim2.new(1, -28, 0, 36); saveConfigBtn.Position = UDim2.new(0, 14, 0, 5)
saveConfigBtn.BackgroundColor3 = Color3.fromRGB(40, 120, 60); saveConfigBtn.BorderSizePixel = 0
saveConfigBtn.Text = "💾  Save Config"; saveConfigBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
saveConfigBtn.Font = Enum.Font.GothamBold; saveConfigBtn.TextSize = 13; saveConfigBtn.ZIndex = 9
mkCorner(saveConfigBtn, 8); mkStroke(saveConfigBtn, Color3.fromRGB(60, 160, 80), 1)
saveConfigBtn.MouseEnter:Connect(function() TweenService:Create(saveConfigBtn,TweenInfo.new(0.1),{BackgroundColor3=Color3.fromRGB(50,150,75)}):Play() end)
saveConfigBtn.MouseLeave:Connect(function() TweenService:Create(saveConfigBtn,TweenInfo.new(0.1),{BackgroundColor3=Color3.fromRGB(40,120,60)}):Play() end)
saveConfigBtn.MouseButton1Click:Connect(function()
    local ok = pcall(saveConfig)
    if ok then
        saveConfigBtn.Text = "✓  Config Saved!"
        TweenService:Create(saveConfigBtn,TweenInfo.new(0.1),{BackgroundColor3=Color3.fromRGB(30,180,80)}):Play()
        task.delay(2, function()
            if saveConfigBtn and saveConfigBtn.Parent then
                saveConfigBtn.Text = "💾  Save Config"
                TweenService:Create(saveConfigBtn,TweenInfo.new(0.3),{BackgroundColor3=Color3.fromRGB(40,120,60)}):Play()
            end
        end)
    else
        saveConfigBtn.Text = "✗  Save Failed"
        task.delay(2, function() if saveConfigBtn and saveConfigBtn.Parent then saveConfigBtn.Text = "💾  Save Config" end end)
    end
end)

makeGap(10)

local fw = Instance.new("Frame", currentPage); fw.Size = UDim2.new(1, 0, 0, 22)
fw.BackgroundTransparency = 1; fw.BorderSizePixel = 0; fw.LayoutOrder = LO()
local fl = Instance.new("TextLabel", fw); fl.Size = UDim2.new(1, 0, 1, 0)
fl.BackgroundTransparency = 1; fl.Text = "Radiant Hub  ·  v6"
fl.TextColor3 = Color3.fromRGB(150, 150, 150); fl.Font = Enum.Font.Gotham; fl.TextSize = 10
fl.TextXAlignment = Enum.TextXAlignment.Center
end)

-- ============================================================
-- REBUILD PRESET LIST UI
-- ============================================================
rebuildPresetList = function()
if not presetListFrame then return end
for _, child in ipairs(presetListFrame:GetChildren()) do
if child.Name ~= "EmptyLabel" and not child:IsA("UIListLayout") and not child:IsA("UIPadding") then
child:Destroy()
end
end
local emptyLbl = presetListFrame:FindFirstChild("EmptyLabel")
if emptyLbl then emptyLbl.Visible = (#Presets == 0) end
for i, preset in ipairs(Presets) do
local row = Instance.new("Frame", presetListFrame)
row.Name = "Preset"..i; row.Size = UDim2.new(1, 0, 0, 34)
row.BackgroundColor3 = C.presetBg; row.BorderSizePixel = 0; row.LayoutOrder = i+1
mkCorner(row, 6); mkStroke(row, C.presetBrd, 1)
local nameLbl = Instance.new("TextLabel", row)
nameLbl.Size = UDim2.new(1, -94, 1, 0); nameLbl.Position = UDim2.new(0, 10, 0, 0)
nameLbl.BackgroundTransparency = 1; nameLbl.Text = preset.name
nameLbl.TextColor3 = C.rowLabel; nameLbl.Font = Enum.Font.GothamBold
nameLbl.TextSize = 12; nameLbl.TextXAlignment = Enum.TextXAlignment.Left
nameLbl.TextTruncate = Enum.TextTruncate.AtEnd
local loadBtn = Instance.new("TextButton", row)
loadBtn.Size = UDim2.new(0, 44, 0, 26); loadBtn.Position = UDim2.new(1, -96, 0.5, -13)
loadBtn.BackgroundColor3 = C.presetLoad; loadBtn.BorderSizePixel = 0
loadBtn.Text = "Load"; loadBtn.TextColor3 = Color3.fromRGB(70,70,70)
loadBtn.Font = Enum.Font.GothamBold; loadBtn.TextSize = 11; loadBtn.ZIndex = 9
mkCorner(loadBtn, 5)
loadBtn.MouseEnter:Connect(function() TweenService:Create(loadBtn,TweenInfo.new(0.1),{BackgroundColor3=Color3.fromRGB(180,160,100)}):Play() end)
loadBtn.MouseLeave:Connect(function() TweenService:Create(loadBtn,TweenInfo.new(0.1),{BackgroundColor3=C.presetLoad}):Play() end)
loadBtn.MouseButton1Click:Connect(function()
applyPreset(preset.data); saveLastPresetName(preset.name)
loadBtn.Text = "✓"; task.delay(1.2, function() if loadBtn and loadBtn.Parent then loadBtn.Text = "Load" end end)
end)
local delBtn = Instance.new("TextButton", row)
delBtn.Size = UDim2.new(0, 34, 0, 26); delBtn.Position = UDim2.new(1, -48, 0.5, -13)
delBtn.BackgroundColor3 = C.presetDel; delBtn.BorderSizePixel = 0
delBtn.Text = "✕"; delBtn.TextColor3 = Color3.fromRGB(100,80,80)
delBtn.Font = Enum.Font.GothamBold; delBtn.TextSize = 11; delBtn.ZIndex = 9
mkCorner(delBtn, 5)
delBtn.MouseEnter:Connect(function() TweenService:Create(delBtn,TweenInfo.new(0.1),{BackgroundColor3=Color3.fromRGB(220,140,140)}):Play() end)
delBtn.MouseLeave:Connect(function() TweenService:Create(delBtn,TweenInfo.new(0.1),{BackgroundColor3=C.presetDel}):Play() end)
delBtn.MouseButton1Click:Connect(function()
table.remove(Presets, i); savePresetsFile(); rebuildPresetList()
end)
end
end

-- Init tab states
for _, n in ipairs(TABS) do
local t = tabBtns[n]; local active = (n == "Speed")
t.btn.TextColor3 = active and C.tabActive or C.tabIdle
t.btn.BackgroundColor3 = active and RED or Color3.fromRGB(255,255,255)
t.btn.BackgroundTransparency = active and 0 or 1
if tabPages[n] then tabPages[n].Visible = active end
end

-- ============================================================
-- VBTN (floating toggle button) — wide pill "RH" style
-- ============================================================
local vBtnFrame = Instance.new("Frame", gui)
vBtnFrame.Name = "RadiantHubVBtn"
vBtnFrame.Size = UDim2.new(0, 130, 0, 34)
vBtnFrame.BackgroundColor3 = Color3.fromRGB(12, 12, 12)
vBtnFrame.BorderSizePixel = 0
vBtnFrame.Active = true; vBtnFrame.ZIndex = 20
mkCorner(vBtnFrame, 10)
mkStroke(vBtnFrame, Color3.fromRGB(45, 45, 45), 1.5)

-- Load saved MH button position, default to left side matching screenshot
local VBTN_POS_FILE = "RadiantHub_vbtn_pos.json"
local function saveVBtnPos()
    local pos = vBtnFrame.Position
    pcall(function()
        _writefile(VBTN_POS_FILE, HttpService:JSONEncode({x=pos.X.Offset, y=pos.Y.Offset}))
    end)
end
do
    local ok, raw = pcall(_readfile, VBTN_POS_FILE)
    if ok and raw then
        local ok2, data = pcall(function() return HttpService:JSONDecode(raw) end)
        if ok2 and data and data.x then
            vBtnFrame.Position = UDim2.new(0, data.x, 0, data.y)
        else
            vBtnFrame.Position = UDim2.new(0, 14, 0, 168) -- left side default
        end
    else
        vBtnFrame.Position = UDim2.new(0, 14, 0, 168) -- left side default
    end
end

local vBtnTxt = Instance.new("TextLabel", vBtnFrame)
vBtnTxt.Size = UDim2.new(1, 0, 1, 0)
vBtnTxt.BackgroundTransparency = 1
vBtnTxt.Text = "RADIANT HUB"
vBtnTxt.Font = Enum.Font.GothamBold
vBtnTxt.TextSize = 14
vBtnTxt.TextColor3 = Color3.fromRGB(220, 220, 220)
vBtnTxt.TextXAlignment = Enum.TextXAlignment.Center
vBtnTxt.TextYAlignment = Enum.TextYAlignment.Center
vBtnTxt.ZIndex = 21

local vDragging,vDragInput,vDragStart,vStartPos=false,nil,nil,nil; local vMoved=false
vBtnFrame.InputBegan:Connect(function(inp)
if inp.UserInputType~=Enum.UserInputType.MouseButton1 and inp.UserInputType~=Enum.UserInputType.Touch then return end
vDragging=true; vMoved=false; vDragStart=inp.Position; vStartPos=vBtnFrame.Position
inp.Changed:Connect(function()
if inp.UserInputState==Enum.UserInputState.End then
if not vMoved then State.guiVisible=not State.guiVisible; mainOuter.Visible=State.guiVisible end
vDragging=false; vMoved=false
if vMoved then saveVBtnPos() end
end
end)
end)
vBtnFrame.InputChanged:Connect(function(inp)
if inp.UserInputType==Enum.UserInputType.MouseMovement or inp.UserInputType==Enum.UserInputType.Touch then vDragInput=inp end
end)
UIS.InputChanged:Connect(function(inp)
if inp~=vDragInput or not vDragging then return end
local dx=inp.Position.X-vDragStart.X; local dy=inp.Position.Y-vDragStart.Y
if math.abs(dx)>4 or math.abs(dy)>4 then vMoved=true end
if vMoved then
    vBtnFrame.Position=UDim2.new(vStartPos.X.Scale,vStartPos.X.Offset+dx,vStartPos.Y.Scale,vStartPos.Y.Offset+dy)
end
end)
UIS.InputEnded:Connect(function(inp)
if (inp.UserInputType==Enum.UserInputType.MouseButton1 or inp.UserInputType==Enum.UserInputType.Touch) and vMoved then
    saveVBtnPos()
end
end)

-- ============================================================
-- STEAL BAR (new — draggable, animated border, ping/fps, progress)
-- ============================================================
local _sbGui = Instance.new("ScreenGui")
_sbGui.Name = "RadiantHubInfoBar"
_sbGui.ResetOnSpawn = false
_sbGui.DisplayOrder = 11
_sbGui.IgnoreGuiInset = true
_sbGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
_sbGui.Parent = LP:WaitForChild("PlayerGui")

-- Outer draggable container
local infoBar = Instance.new("Frame", _sbGui)
infoBar.Name = "StealBarContainer"
infoBar.Size = UDim2.new(0, 260, 0, 70)
infoBar.Position = UDim2.new(0.5, -130, 0, 35)
infoBar.BackgroundTransparency = 1
infoBar.BorderSizePixel = 0
infoBar.Active = true

-- Banner (top pill — Ping / FPS)
local _bannerFrame = Instance.new("Frame", infoBar)
_bannerFrame.Size = UDim2.new(1, 0, 0, 30)
_bannerFrame.Position = UDim2.new(0, 0, 0, 0)
_bannerFrame.BackgroundColor3 = Color3.fromRGB(0, 255, 100)
_bannerFrame.BackgroundTransparency = 0.85
_bannerFrame.BorderSizePixel = 0
Instance.new("UICorner", _bannerFrame).CornerRadius = UDim.new(0, 8)
local _bannerStroke = Instance.new("UIStroke", _bannerFrame)
_bannerStroke.Color = Color3.fromRGB(0, 255, 100)
_bannerStroke.Thickness = 1.5

local _infoLabel = Instance.new("TextLabel", _bannerFrame)
_infoLabel.Size = UDim2.new(1, 0, 1, 0)
_infoLabel.BackgroundTransparency = 1
_infoLabel.Font = Enum.Font.GothamBold
_infoLabel.TextSize = 14
_infoLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
_infoLabel.Text = "Radiant Hub | Ping: 0ms | FPS: 0"
_infoLabel.TextXAlignment = Enum.TextXAlignment.Center

-- Progress bar (bottom pill)
local _progressBarBg = Instance.new("Frame", infoBar)
_progressBarBg.Size = UDim2.new(1, 0, 0, 14)
_progressBarBg.Position = UDim2.new(0, 0, 0, 34)
_progressBarBg.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
_progressBarBg.BackgroundTransparency = 0.25
_progressBarBg.BorderSizePixel = 0
Instance.new("UICorner", _progressBarBg).CornerRadius = UDim.new(0, 10)
local _bgStroke = Instance.new("UIStroke", _progressBarBg)
_bgStroke.Color = Color3.fromRGB(0, 255, 100)
_bgStroke.Thickness = 1

local progressFill = Instance.new("Frame", _progressBarBg)
progressFill.Size = UDim2.new(0, 0, 1, 0)
progressFill.BackgroundColor3 = Color3.fromRGB(34, 197, 94)
progressFill.BorderSizePixel = 0
Instance.new("UICorner", progressFill).CornerRadius = UDim.new(0, 10)

local stealPctLbl = Instance.new("TextLabel", _progressBarBg)
stealPctLbl.Size = UDim2.new(1, 0, 1, 0)
stealPctLbl.BackgroundTransparency = 1
stealPctLbl.Font = Enum.Font.GothamBold
stealPctLbl.TextSize = 11
stealPctLbl.TextColor3 = Color3.fromRGB(220, 220, 220)
stealPctLbl.Text = "0%"
stealPctLbl.ZIndex = 2

-- ── Animated border color cycling (3 greens) ─────────────────
do
    local _borderColors = {
        Color3.fromRGB(0, 255, 100),
        Color3.fromRGB(0, 180, 60),
        Color3.fromRGB(57, 255, 180),
    }
    local _bi = 1
    local _bt = 0
    RunService.RenderStepped:Connect(function(dt)
        _bt = _bt + dt
        if _bt >= 0.6 then
            _bt = 0
            _bi = (_bi % #_borderColors) + 1
            local col = _borderColors[_bi]
            _bannerStroke.Color = col
            _bannerFrame.BackgroundColor3 = col
            _bgStroke.Color = col
        end
    end)
end

-- ── FPS + Ping live update ────────────────────────────────────
do
    local _fpsFrames = 0
    local _fpsLast = tick()
    local _fps = 0
    RunService.RenderStepped:Connect(function()
        _fpsFrames = _fpsFrames + 1
        local now = tick()
        if now - _fpsLast >= 1 then
            _fps = _fpsFrames
            _fpsFrames = 0
            _fpsLast = now
        end
    end)
    task.spawn(function()
        while task.wait(1) do
            pcall(function()
                local ping = math.floor(Stats.Network.ServerStatsItem["Data Ping"]:GetValue())
                _infoLabel.Text = string.format("Radiant Hub | Ping: %dms | FPS: %d", ping, _fps)
            end)
        end
    end)
end

-- ── Drag ─────────────────────────────────────────────────────
do
    local _dragging, _dragStart, _startPos = false, nil, nil
    local _moved = false
    infoBar.InputBegan:Connect(function(inp)
        if inp.UserInputType ~= Enum.UserInputType.MouseButton1
        and inp.UserInputType ~= Enum.UserInputType.Touch then return end
        if State.uiLocked then return end
        _dragging = true; _moved = false
        _dragStart = inp.Position; _startPos = infoBar.Position
        inp.Changed:Connect(function()
            if inp.UserInputState == Enum.UserInputState.End then
                _dragging = false
                if _moved then task.spawn(function() pcall(saveConfig) end) end
                _moved = false
            end
        end)
    end)
    UIS.InputChanged:Connect(function(inp)
        if not _dragging then return end
        if inp.UserInputType ~= Enum.UserInputType.MouseMovement
        and inp.UserInputType ~= Enum.UserInputType.Touch then return end
        local dx = inp.Position.X - _dragStart.X
        local dy = inp.Position.Y - _dragStart.Y
        if math.abs(dx) > 3 or math.abs(dy) > 3 then _moved = true end
        if _moved then
            infoBar.Position = UDim2.new(
                _startPos.X.Scale, _startPos.X.Offset + dx,
                _startPos.Y.Scale, _startPos.Y.Offset + dy)
        end
    end)
end




-- ============================================================
-- AUTO BAT (AIMBOT) LOGIC  –  ported from CursedHub AutoBat
-- ============================================================
local AUTO_BAT_SPEED      =  58
local AUTO_BAT_VERT_SPEED =  52
local AUTO_BAT_DIST       =  -2.8
local AUTO_BAT_HEIGHT     =   4.75
local AUTO_BAT_V_OFF      =   1
local AUTO_BAT_TURN_SPEED =  285
local AUTO_BAT_MAX_TURN_RATE = 28

local _abEnabled          = false
local _abEquippedThisRun  = false  -- kept to avoid breaking references, unused now
local _abTarget           = nil
local _abLastScan         = 0
local _abSwingDebounce    = false

local function _abGetTarget()
    local root = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
    if not root then return nil end
    local now = tick()
    if now - _abLastScan <= 0.1 and _abTarget and _abTarget.Parent then
        local hum2 = _abTarget.Parent:FindFirstChildOfClass("Humanoid")
        if hum2 and hum2.Health > 0 then return _abTarget end
    end
    _abLastScan = now
    _abTarget   = nil
    local closest, minDist = nil, math.huge
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LP and plr.Character then
            local tRoot = plr.Character:FindFirstChild("HumanoidRootPart")
            local hum2  = plr.Character:FindFirstChildOfClass("Humanoid")
            if tRoot and hum2 and hum2.Health > 0 then
                local dist = (tRoot.Position - root.Position).Magnitude
                if dist < minDist then minDist = dist; closest = tRoot end
            end
        end
    end
    _abTarget = closest
    return _abTarget
end

local function _abResetMotion()
    local char = LP.Character
    local hrp2  = char and char:FindFirstChild("HumanoidRootPart")
    local hum2  = char and char:FindFirstChildOfClass("Humanoid")
    if hrp2 then
        hrp2.AssemblyLinearVelocity  = hrp2.AssemblyLinearVelocity * 0.3
        hrp2.AssemblyAngularVelocity = Vector3.zero
    end
    if hum2 then hum2.AutoRotate = true end
end

local _abPrevTpDown = false  -- remembers Auto TP Down state before aimbot was enabled

local function enableAutoBat()
    _abEnabled         = true
    _abSwingDebounce   = false
    -- save and suppress Auto TP Down while aimbot is active
    _abPrevTpDown = State.autoTpDownEnabled
    State.autoTpDownEnabled = false
    if setAutoTpDown then setAutoTpDown(false) end
end

local function disableAutoBat()
    _abEnabled        = false
    _abSwingDebounce  = false
    local char = LP.Character
    if char then
        local hum2 = char:FindFirstChildOfClass("Humanoid")
        if hum2 then hum2.AutoRotate = true end
    end
    _abResetMotion()
    -- restore Auto TP Down to whatever it was before aimbot was turned on
    State.autoTpDownEnabled = _abPrevTpDown
    if setAutoTpDown then setAutoTpDown(_abPrevTpDown) end
end

-- Heartbeat: movement + swing
RunService.Heartbeat:Connect(function()
    if not _abEnabled then return end
    local char = LP.Character
    local hum2  = char and char:FindFirstChildOfClass("Humanoid")
    local root2 = char and char:FindFirstChild("HumanoidRootPart")
    if not root2 or not hum2 then return end

    local target = _abGetTarget()
    if target then
        local targetVel    = target.AssemblyLinearVelocity
        local leadFactor   = math.clamp(targetVel.Magnitude / 130, 0.05, 0.15)
        local aimTargetPos = target.Position + (targetVel * leadFactor) + Vector3.new(0, AUTO_BAT_V_OFF, 0)

        -- rotation / aiming
        hum2.AutoRotate = false
        local look     = aimTargetPos - root2.Position
        local flatLook = Vector3.new(look.X, 0, look.Z)

        if look.Magnitude > 0.01 and flatLook.Magnitude > 0.01 then
            local targetYaw   = math.deg(math.atan2(-flatLook.X, -flatLook.Z))
            local yawDelta    = (targetYaw - root2.Orientation.Y + 180) % 360 - 180
            local targetPitch = math.deg(math.atan2(look.Y, flatLook.Magnitude))
            local pitchDelta  = (targetPitch - root2.Orientation.X + 180) % 360 - 180

            local yawRate   = math.clamp(math.rad(yawDelta)   * AUTO_BAT_TURN_SPEED, -AUTO_BAT_MAX_TURN_RATE, AUTO_BAT_MAX_TURN_RATE)
            local pitchRate = math.clamp(math.rad(pitchDelta) * AUTO_BAT_TURN_SPEED, -AUTO_BAT_MAX_TURN_RATE, AUTO_BAT_MAX_TURN_RATE)
            local yawRad    = math.rad(root2.Orientation.Y)
            local rightAxis = Vector3.new(math.cos(yawRad), 0, -math.sin(yawRad))
            root2.AssemblyAngularVelocity = Vector3.new(0, yawRate, 0) + (rightAxis * pitchRate)
        else
            root2.AssemblyAngularVelocity = Vector3.zero
        end

        -- position / movement
        local dir      = look.Magnitude > 0.01 and look.Unit or Vector3.zero
        local standPos = aimTargetPos - (dir * AUTO_BAT_DIST) + Vector3.new(0, AUTO_BAT_HEIGHT, 0)
        local moveDir  = standPos - root2.Position
        local hDir     = Vector3.new(moveDir.X, 0, moveDir.Z)

        local hVel = hDir.Magnitude > 0.1
            and hDir.Unit * AUTO_BAT_SPEED
            or  Vector3.zero
        local vVel = math.abs(moveDir.Y) > 0.1
            and Vector3.new(0, math.sign(moveDir.Y) * AUTO_BAT_VERT_SPEED, 0)
            or  Vector3.new(0, -2, 0)

        root2.AssemblyLinearVelocity = hVel + vVel
        if hDir.Magnitude > 0.5 then hum2:Move(hDir.Unit, false) end
    else
        hum2.AutoRotate               = true
        root2.AssemblyAngularVelocity = Vector3.zero
    end

    -- cursed auto swing: FireServer on RemoteEvent, fallback to Activate
    if State.autoSwingEnabled and not _abSwingDebounce then
        local bat = char:FindFirstChildOfClass("Tool")
        if bat then
            _abSwingDebounce = true
            task.spawn(function()
                local remote = bat:FindFirstChildOfClass("RemoteEvent") or bat:FindFirstChildOfClass("RemoteFunction")
                if remote and remote:IsA("RemoteEvent") then
                    pcall(function() remote:FireServer() end)
                    task.wait(0.15)
                    pcall(function() remote:FireServer() end)
                else
                    pcall(function() bat:Activate() end)
                    task.wait(0.15)
                    pcall(function() bat:Activate() end)
                end
                task.wait(0.1)
                _abSwingDebounce = false
            end)
        end
    end
end)

-- ============================================================
-- STACK BUTTONS
-- ============================================================
for i, def in ipairs(stackDefs) do
local btnFrame = Instance.new("Frame", gui)
btnFrame.Name = "StackBtn_"..def.key
btnFrame.Size = UDim2.new(0, BTN_W, 0, BTN_H)
btnFrame.Position = getDefaultStackPos(i)
btnFrame.BackgroundColor3 = Color3.fromRGB(10, 10, 10)
btnFrame.BackgroundTransparency = 0; btnFrame.BorderSizePixel = 0
btnFrame.Active = true; btnFrame.ZIndex = 2
local _btnCorner = Instance.new("UICorner", btnFrame)
_btnCorner.CornerRadius = UDim.new(0, 14)
-- Stroke: thin dark idle, animated light green chaser when active
local bStroke = Instance.new("UIStroke", btnFrame)
bStroke.Thickness = 1.5
bStroke.Color = Color3.fromRGB(50, 50, 50)
bStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
stackWrappers[def.key] = btnFrame

-- Bold white text, centered
local nl = Instance.new("TextLabel", btnFrame)
nl.Size = UDim2.new(1, -6, 1, -6); nl.Position = UDim2.new(0, 3, 0, 3)
nl.BackgroundTransparency = 1; nl.Text = def.label; nl.TextColor3 = Color3.fromRGB(255, 255, 255)
nl.Font = Enum.Font.GothamBlack; nl.TextSize = 15; nl.TextWrapped = true
nl.TextXAlignment = Enum.TextXAlignment.Center
nl.TextYAlignment = Enum.TextYAlignment.Center; nl.ZIndex = 3

local dot = Instance.new("Frame", btnFrame)
dot.Size = UDim2.new(0, 0, 0, 0); dot.BackgroundTransparency = 1; dot.BorderSizePixel = 0

local btnState = false
local chaserConn = nil
local chaserPhase = 0

-- Light green chaser gradient on the stroke border
local chaserGrad = Instance.new("UIGradient", bStroke)
chaserGrad.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0,    Color3.fromRGB(10, 40, 10)),
    ColorSequenceKeypoint.new(0.12, Color3.fromRGB(10, 40, 10)),
    ColorSequenceKeypoint.new(0.25, Color3.fromRGB(30, 120, 30)),
    ColorSequenceKeypoint.new(0.38, Color3.fromRGB(80, 200, 60)),
    ColorSequenceKeypoint.new(0.50, Color3.fromRGB(150, 255, 80)),
    ColorSequenceKeypoint.new(0.62, Color3.fromRGB(80, 200, 60)),
    ColorSequenceKeypoint.new(0.75, Color3.fromRGB(30, 120, 30)),
    ColorSequenceKeypoint.new(0.88, Color3.fromRGB(10, 40, 10)),
    ColorSequenceKeypoint.new(1,    Color3.fromRGB(10, 40, 10)),
}
chaserGrad.Rotation = 0

local function startChaser()
    if chaserConn then return end
    chaserConn = RunService.RenderStepped:Connect(function(dt)
        chaserPhase = (chaserPhase + dt * 130) % 360
        chaserGrad.Rotation = chaserPhase
    end)
end
local function stopChaser()
    if chaserConn then chaserConn:Disconnect(); chaserConn = nil end
    chaserPhase = 0; chaserGrad.Rotation = 0
end

local function setOn(on)
    btnState = on
    -- Active: dark green bg, chaser running. Idle: black bg, no chaser
    TweenService:Create(btnFrame, TweenInfo.new(0.15), {
        BackgroundColor3 = on and Color3.fromRGB(10, 50, 10) or Color3.fromRGB(10, 10, 10),
        BackgroundTransparency = 0
    }):Play()
    -- Active: chaser border (thick). Idle: solid thin dark border
    TweenService:Create(bStroke, TweenInfo.new(0.15), {
        Thickness = on and 2.5 or 1.5,
        Color = on and Color3.fromRGB(80, 200, 60) or Color3.fromRGB(50, 50, 50)
    }):Play()
    TweenService:Create(nl, TweenInfo.new(0.15), {TextColor3 = Color3.fromRGB(255, 255, 255)}):Play()
    if on then startChaser() else stopChaser() end
end
stackBtnRefs[def.key] = {setOn = setOn}

-- Stop chaser immediately — buttons are idle (unpressed) by default
stopChaser()

btnFrame.MouseEnter:Connect(function()
    if not btnState then TweenService:Create(btnFrame,TweenInfo.new(0.1),{BackgroundColor3=Color3.fromRGB(20,20,20)}):Play() end
end)
btnFrame.MouseLeave:Connect(function()
    TweenService:Create(btnFrame,TweenInfo.new(0.1),{BackgroundColor3=btnState and Color3.fromRGB(10,50,10) or Color3.fromRGB(10,10,10)}):Play()
end)

local function onTap()
    if def.key == "tpDown" then doTpDown(); return end
    if def.key == "carrySpeed" then State.speedToggled = not State.speedToggled; setOn(State.speedToggled); return end
    if def.key == "taunt" then
        pcall(function()
            local msg = "Radiant OWNS"
            -- Method 1: TextChatService (new Roblox chat)
            local tcs = game:GetService("TextChatService")
            if tcs and tcs.TextChannels then
                local ch = tcs.TextChannels:FindFirstChild("RBXGeneral")
                if ch then ch:SendAsync(msg) return end
            end
            -- Method 2: Legacy SayMessageRequest
            local rs = game:GetService("ReplicatedStorage")
            local events = rs:FindFirstChild("DefaultChatSystemChatEvents")
                or rs:FindFirstChildWhichIsA("Folder")
            if events then
                local sayMsg = events:FindFirstChild("SayMessageRequest")
                if sayMsg then sayMsg:FireServer(msg, "All") return end
            end
            -- Method 3: FindFirstChild anywhere
            for _, v in pairs(rs:GetDescendants()) do
                if v.Name == "SayMessageRequest" and v:IsA("RemoteEvent") then
                    v:FireServer(msg, "All") return
                end
            end
        end)
        -- Visual flash feedback
        TweenService:Create(btnFrame, TweenInfo.new(0.1), {BackgroundColor3 = Color3.fromRGB(0, 160, 60)}):Play()
        task.delay(0.25, function()
            TweenService:Create(btnFrame, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(10, 10, 10)}):Play()
        end)
        return
    end
    local ns = not btnState; setOn(ns)
    if def.key == "autoLeft" then
        State.autoLeftEnabled = ns
        if ns then startAutoLeft() else stopAutoLeft() end
    elseif def.key == "autoRight" then
        State.autoRightEnabled = ns
        if ns then startAutoRight() else stopAutoRight() end
    elseif def.key == "aimbot" then
        State.batAimbotToggled = ns
        if ns then enableAutoBat() else disableAutoBat() end
    elseif def.key == "lagger" then
        State.laggerEnabled = ns
        if ns then
            State._prevCarry = State.carrySpeed
            State._prevSpeed = State.speedToggled
            State.speedToggled = false
            if stackBtnRefs.carrySpeed then stackBtnRefs.carrySpeed.setOn(false) end
            if carryBox then carryBox.Text = tostring(State.laggerSpeed) end
        else
            State.carrySpeed = State._prevCarry or 30
            State.speedToggled = State._prevSpeed or false
            if carryBox then carryBox.Text = tostring(State.carrySpeed) end
            if stackBtnRefs.carrySpeed then stackBtnRefs.carrySpeed.setOn(State.speedToggled) end
        end
    elseif def.key == "drop" then
        if ns then runDropBrainrot() else stopDropBrainrot() end
    end
end
makeStackDraggable(btnFrame, onTap)
end

-- ============================================================
-- HELPERS
-- ============================================================
local resetProgressBar = WYNF_NO_VIRTUALIZE(function() stealPctLbl.Text="0%"; progressFill.Size=UDim2.new(0,0,1,0) end)

doTpDown = function()
pcall(function()
local c = LP.Character; if not c then return end
local root = c:FindFirstChild("HumanoidRootPart"); if not root then return end
local rot = root.CFrame.Rotation
local rp = RaycastParams.new()
rp.FilterType = Enum.RaycastFilterType.Exclude
rp.FilterDescendantsInstances = {c}
local res = workspace:Raycast(root.Position, Vector3.new(0, -500, 0), rp)
if res then
    root.CFrame = CFrame.new(res.Position + Vector3.new(0, 3, 0)) * rot
else
    root.CFrame = CFrame.new(root.Position + Vector3.new(0, -20, 0)) * rot
end
end)
end

-- ============================================================
-- DROP BRAINROT
-- ============================================================
local _dropConns={}
runDropBrainrot=function()
if State.dropEnabled then return end; State.dropEnabled=true
if stackBtnRefs.drop then stackBtnRefs.drop.setOn(true) end
task.spawn(function()
-- ── No-reset shield ────────────────────────────────────────────
local _dropConnsLocal = {}
local _dropHum = nil
local _savedMaxHealth = 100

local DEAD_STATES = {
	Enum.HumanoidStateType.Dead,
	Enum.HumanoidStateType.FallingDown,
	Enum.HumanoidStateType.Ragdoll,
	Enum.HumanoidStateType.Physics,
}

local function applyShield(c)
	if not c then return end
	local hum2 = c:FindFirstChildOfClass("Humanoid")
	if not hum2 then return end
	_dropHum = hum2
	_savedMaxHealth = hum2.MaxHealth

	-- 1. Disable all death-leading states
	for _, st in ipairs(DEAD_STATES) do
		pcall(function() hum2:SetStateEnabled(st, false) end)
	end

	-- 2. Block BreakJoints on every BasePart (prevents server killing character)
	for _, part in ipairs(c:GetDescendants()) do
		if part:IsA("BasePart") then
			pcall(function()
				part:SetNetworkOwner(LP)
			end)
		end
	end

	-- 3. Keep health at max every heartbeat
	local hbConn; hbConn = RunService.Heartbeat:Connect(function()
		if not State.dropEnabled then hbConn:Disconnect(); return end
		pcall(function()
			if hum2 and hum2.Parent then
				if hum2.Health < _savedMaxHealth then hum2.Health = _savedMaxHealth end
			end
		end)
	end)
	table.insert(_dropConnsLocal, hbConn)

	-- 4. If Died fires anyway, immediately restore health so the game can't respawn
	local diedConn; diedConn = hum2.Died:Connect(function()
		pcall(function()
			hum2.Health = _savedMaxHealth
			for _, st in ipairs(DEAD_STATES) do
				pcall(function() hum2:SetStateEnabled(st, false) end)
			end
		end)
	end)
	table.insert(_dropConnsLocal, diedConn)

	-- 5. Snap out of any bad state every heartbeat
	local stateConn; stateConn = RunService.Heartbeat:Connect(function()
		if not State.dropEnabled then stateConn:Disconnect(); return end
		pcall(function()
			if hum2 and hum2.Parent then
				local st = hum2:GetState()
				if st == Enum.HumanoidStateType.Dead
				or st == Enum.HumanoidStateType.FallingDown
				or st == Enum.HumanoidStateType.Ragdoll then
					hum2:ChangeState(Enum.HumanoidStateType.Running)
				end
			end
		end)
	end)
	table.insert(_dropConnsLocal, stateConn)
end

local function removeShield()
	for _, cn in ipairs(_dropConnsLocal) do pcall(function() cn:Disconnect() end) end
	_dropConnsLocal = {}
	if _dropHum and _dropHum.Parent then
		for _, st in ipairs(DEAD_STATES) do
			pcall(function() _dropHum:SetStateEnabled(st, true) end)
		end
		_dropHum = nil
	end
end

applyShield(LP.Character)
-- ── Drop velocity loop ──────────────────────────────────────────
-- No separate Stepped collision loop — the global throttled loop already handles it
task.spawn(function()
while State.dropEnabled do
RunService.Heartbeat:Wait()
local c=LP.Character; local root=c and c:FindFirstChild("HumanoidRootPart")
if not root then continue end
local vel=root.Velocity
root.Velocity=vel*10000+Vector3.new(0,10000,0)
RunService.RenderStepped:Wait()
if root and root.Parent then root.Velocity=vel end
RunService.Stepped:Wait()
if root and root.Parent then root.Velocity=vel+Vector3.new(0,0.1,0) end
end
end)
task.wait(DROP_AUTO_OFF_DELAY); stopDropBrainrot()
removeShield()
end)
end
stopDropBrainrot=function()
State.dropEnabled=false
for _,cn in ipairs(_dropConns) do pcall(function() cn:Disconnect() end) end; _dropConns={}
if stackBtnRefs.drop then stackBtnRefs.drop.setOn(false) end
end

-- ============================================================
-- ============================================================
-- BAT COUNTER
-- ============================================================
local BAT_COUNTER_SLAP_LIST={"Bat","Slap","Iron Slap","Gold Slap","Diamond Slap","Emerald Slap","Ruby Slap","Dark Matter Slap","Flame Slap","Nuclear Slap","Galaxy Slap","Glitched Slap"}

local function findBatForCounter()
local c=LP.Character; if not c then return nil end
local bp=LP:FindFirstChildOfClass("Backpack")
for _,name in ipairs(BAT_COUNTER_SLAP_LIST) do
local t=c:FindFirstChild(name) or (bp and bp:FindFirstChild(name))
if t then return t end
end
for _,ch in ipairs(c:GetChildren()) do if ch:IsA("Tool") and ch.Name:lower():find("bat") then return ch end end
if bp then for _,ch in ipairs(bp:GetChildren()) do if ch:IsA("Tool") and ch.Name:lower():find("bat") then return ch end end end
return nil
end

local function swingBatForCounter(bat,char)
local hum2=char:FindFirstChildOfClass("Humanoid")
if bat.Parent~=char then if hum2 then pcall(function() hum2:EquipTool(bat) end) end; task.wait(0.05) end
local remote=bat:FindFirstChildOfClass("RemoteEvent") or bat:FindFirstChildOfClass("RemoteFunction")
if remote and remote:IsA("RemoteEvent") then
pcall(function() remote:FireServer() end); task.wait(0.15); pcall(function() remote:FireServer() end)
else pcall(function() bat:Activate() end); task.wait(0.15); pcall(function() bat:Activate() end) end
end

startBatCounter=function()
if Conns.batCounter then return end
Conns.batCounter=RunService.Heartbeat:Connect(function()
if not State.batCounterEnabled then return end
if State.batCounterDebounce then return end
local char=LP.Character; if not char then return end
local hum2=char:FindFirstChildOfClass("Humanoid"); if not hum2 then return end
local st=hum2:GetState()
local isRagdolled=st==Enum.HumanoidStateType.Physics or st==Enum.HumanoidStateType.Ragdoll or st==Enum.HumanoidStateType.FallingDown
if isRagdolled then
State.batCounterDebounce=true
task.spawn(function()
local bat=findBatForCounter()
if bat then swingBatForCounter(bat,char) end
task.wait(0.5); State.batCounterDebounce=false
end)
end
end)
end
stopBatCounter=function()
if Conns.batCounter then Conns.batCounter:Disconnect(); Conns.batCounter=nil end
State.batCounterDebounce=false
end

-- ============================================================
-- MEDUSA
-- ============================================================
local function findMedusa()
local c=LP.Character; if not c then return nil end
for _,t in ipairs(c:GetChildren()) do if t:IsA("Tool") then local n=t.Name:lower(); if n:find("medusa") or n:find("head") or n:find("stone") then return t end end end
local bp=LP:FindFirstChild("Backpack")
if bp then for _,t in ipairs(bp:GetChildren()) do if t:IsA("Tool") then local n=t.Name:lower(); if n:find("medusa") or n:find("head") or n:find("stone") then return t end end end end
return nil
end
local function useMedusaCounter()
if State.medusaDebounce then return end; if tick()-State.medusaLastUsed<MEDUSA_COOLDOWN then return end
local c=LP.Character; if not c then return end; State.medusaDebounce=true
local med=findMedusa(); if not med then State.meduzaDebounce=false; return end
if med.Parent~=c then local hum2=c:FindFirstChildOfClass("Humanoid"); if hum2 then hum2:EquipTool(med) end end
pcall(function() med:Activate() end); State.medusaLastUsed=tick(); State.medusaDebounce=false
end
local function onAnchorChanged(part) return part:GetPropertyChangedSignal("Anchored"):Connect(function() if part.Anchored and part.Transparency==1 then useMedusaCounter() end end) end
setupMedusaCounter=function(char)
stopMedusaCounter(); if not char then return end
for _,part in ipairs(char:GetDescendants()) do if part:IsA("BasePart") then table.insert(Conns.anchor,onAnchorChanged(part)) end end
table.insert(Conns.anchor,char.DescendantAdded:Connect(function(part) if part:IsA("BasePart") then table.insert(Conns.anchor,onAnchorChanged(part)) end end))
end
stopMedusaCounter=function() for _,c2 in pairs(Conns.anchor) do pcall(function() c2:Disconnect() end) end; Conns.anchor={} end

-- ============================================================
-- AUTO LEFT / RIGHT (Mace Hub v6 fix39)
-- ============================================================
local faceSouth = WYNF_NO_VIRTUALIZE(function() pcall(function() local c=LP.Character; if not c then return end; local root=c:FindFirstChild("HumanoidRootPart"); if root then root.CFrame=CFrame.new(root.Position)*CFrame.Angles(0,0,0) end end) end)
local faceNorth = WYNF_NO_VIRTUALIZE(function() pcall(function() local c=LP.Character; if not c then return end; local root=c:FindFirstChild("HumanoidRootPart"); if root then root.CFrame=CFrame.new(root.Position)*CFrame.Angles(0,math.rad(180),0) end end) end)

-- ── Misco Hub Full Auto Play Logic ───────────────────────────
local PathfindingService = game:GetService("PathfindingService")
local AP_PLOT3_POS = Vector3.new(-476.7524719238281, 10.464664459228516, 7.107429504394531)
local AP_PLOT7_POS = Vector3.new(-476.7524719238281, 10.464664459228516, 114.10742950439453)
local _apRoutes = {
    START_A = {
        StartPoint = Vector3.new(-484.37,-5.22,94.32),
        Route = {
            Vector3.new(-476,-8,29),
            Vector3.new(-480,-6,25),
            Vector3.new(-486,-6,25),
            Vector3.new(-475.75,-7.03,94.03)
        }
    },
    START_B = {
        StartPoint = Vector3.new(-485.08,-5.22,25.94),
        Route = {
            Vector3.new(-476.07,-8,91.05),
            Vector3.new(-480.07,-6,95.05),
            Vector3.new(-486.07,-6,95.05),
            Vector3.new(-476.08,-6.89,25.65)
        }
    }
}
local _apRouteRunning=false
local _apRouteId=0

local function _apGetMyPlot()
    local char=LP.Character; if not char then return nil end
    local root=char:FindFirstChild("HumanoidRootPart"); if not root then return nil end
    local checks={{plotNum=3,pos=AP_PLOT3_POS},{plotNum=7,pos=AP_PLOT7_POS}}
    for _,entry in ipairs(checks) do
        for _,obj in ipairs(workspace:GetDescendants()) do
            if obj:IsA("BasePart") and obj.Name=="PlotSign" and (obj.Position-entry.pos).Magnitude<5 then
                for _,child in ipairs(obj:GetDescendants()) do
                    if child:IsA("SurfaceGui") then
                        for _,label in ipairs(child:GetDescendants()) do
                            if label:IsA("TextLabel") and label.Text~="" and
                               (string.find(label.Text,LP.Name) or string.find(label.Text,LP.DisplayName)) then
                                return entry.plotNum
                            end
                        end
                    end
                end
            end
        end
    end
    return nil
end

local function _apDirectMove(point, myRouteId, isFast)
    while _apRouteRunning and _apRouteId==myRouteId do
        local char=LP.Character
        local root=char and char:FindFirstChild("HumanoidRootPart")
        local hum=char and char:FindFirstChildOfClass("Humanoid")
        if not root or not hum then task.wait(0.5); continue end
        local spd = State.laggerEnabled and State.laggerSpeed or (isFast and State.normalSpeed or State.carrySpeed)
        State._autoPlaySpeed = spd
        local dir=point-root.Position
        dir=Vector3.new(dir.X,0,dir.Z)
        if dir.Magnitude<2 then break end
        dir=dir.Unit
        hum.WalkSpeed=spd
        hum:Move(dir,false)
        root.AssemblyLinearVelocity=Vector3.new(dir.X*spd,root.AssemblyLinearVelocity.Y,dir.Z*spd)
        task.wait()
    end
    local char=LP.Character
    local root=char and char:FindFirstChild("HumanoidRootPart")
    local hum=char and char:FindFirstChildOfClass("Humanoid")
    if hum then hum:Move(Vector3.zero,false) end
    if root then root.AssemblyLinearVelocity=Vector3.new(0,root.AssemblyLinearVelocity.Y,0) end
end

local function _apPathMove(point)
    local char=LP.Character; if not char then return end
    local root=char:FindFirstChild("HumanoidRootPart"); if not root then return end
    local path=PathfindingService:CreatePath({AgentRadius=2,AgentHeight=5,AgentCanJump=false})
    pcall(function() path:ComputeAsync(root.Position,point) end)
    local waypoints=path.Status==Enum.PathStatus.Success and path:GetWaypoints() or {{Position=point}}
    local index=2
    while _apRouteRunning and index<=#waypoints do
        local c=LP.Character
        local r=c and c:FindFirstChild("HumanoidRootPart")
        local hum=c and c:FindFirstChildOfClass("Humanoid")
        if not r or not hum then task.wait(0.5); continue end
        local spd=State.laggerEnabled and State.laggerSpeed or State.normalSpeed
        State._autoPlaySpeed = spd
        local target=waypoints[index].Position
        local dir=target-r.Position
        dir=Vector3.new(dir.X,0,dir.Z)
        if dir.Magnitude<2 then
            index=index+1
        else
            dir=dir.Unit
            hum.WalkSpeed=spd
            hum:Move(dir,false)
            r.AssemblyLinearVelocity=Vector3.new(dir.X*spd,r.AssemblyLinearVelocity.Y,dir.Z*spd)
        end
        task.wait()
    end
    local c=LP.Character
    local r=c and c:FindFirstChild("HumanoidRootPart")
    local hum=c and c:FindFirstChildOfClass("Humanoid")
    if hum then hum:Move(Vector3.zero,false) end
    if r then r.AssemblyLinearVelocity=Vector3.new(0,r.AssemblyLinearVelocity.Y,0) end
end

local function _apWalkRoute(route,onDone)
    local myId=_apRouteId; _apRouteRunning=true
    _apPathMove(route.Route[1]); if myId~=_apRouteId then _apRouteRunning=false; return end
    _apDirectMove(route.Route[2],myId,true); if myId~=_apRouteId then _apRouteRunning=false; return end
    _apDirectMove(route.Route[3],myId,true); if myId~=_apRouteId then _apRouteRunning=false; return end
    task.wait(0.1)
    _apDirectMove(route.Route[2],myId,false); if myId~=_apRouteId then _apRouteRunning=false; return end
    _apDirectMove(route.Route[1],myId,false); if myId~=_apRouteId then _apRouteRunning=false; return end
    if route.Route[4] then _apDirectMove(route.Route[4],myId,false) end
    _apDirectMove(route.StartPoint,myId,false)
    _apRouteRunning=false
    if onDone then onDone() end
end

startAutoLeft=function()
if Conns.autoLeft then Conns.autoLeft:Disconnect(); Conns.autoLeft=nil end
if modeToggleState=="full" then
_apRouteRunning=false; _apRouteId=_apRouteId+1
task.spawn(function()
local plot=_apGetMyPlot()
local detectedSpawn=plot==3 and 1 or (plot==7 and 2 or nil)
local route=detectedSpawn==1 and _apRoutes.START_A or _apRoutes.START_B
_apWalkRoute(route,function()
State.autoLeftEnabled=false
if stackBtnRefs.autoLeft then stackBtnRefs.autoLeft.setOn(false) end
-- Animal delivered — turn carry speed back off (full auto only)
if modeToggleState == "full" then
    State.speedToggled = false
    if stackBtnRefs.carrySpeed then stackBtnRefs.carrySpeed.setOn(false) end
end
end)
end)
else
State.autoLeftPhase=1
Conns.autoLeft=RunService.Heartbeat:Connect(function()
if not State.autoLeftEnabled then return end
local c=LP.Character; if not c then return end
local root=c:FindFirstChild("HumanoidRootPart"); local hum2=c:FindFirstChildOfClass("Humanoid"); if not root or not hum2 then return end
local spd=State.laggerEnabled and State.laggerSpeed or State.normalSpeed
State._autoPlaySpeed = spd
hum2.WalkSpeed = spd
if State.autoLeftPhase==1 then
local tgt=Vector3.new(POS.L1.X,root.Position.Y,POS.L1.Z); if (tgt-root.Position).Magnitude<1 then State.autoLeftPhase=2; local d=(POS.L2-root.Position); local mv=Vector3.new(d.X,0,d.Z).Unit; hum2:Move(mv,false); root.AssemblyLinearVelocity=Vector3.new(mv.X*spd,root.AssemblyLinearVelocity.Y,mv.Z*spd); return end
local d=(POS.L1-root.Position); local mv=Vector3.new(d.X,0,d.Z).Unit; hum2:Move(mv,false); root.AssemblyLinearVelocity=Vector3.new(mv.X*spd,root.AssemblyLinearVelocity.Y,mv.Z*spd)
elseif State.autoLeftPhase==2 then
local tgt=Vector3.new(POS.L2.X,root.Position.Y,POS.L2.Z); if (tgt-root.Position).Magnitude<1 then hum2:Move(Vector3.zero,false); root.AssemblyLinearVelocity=Vector3.zero; State.autoLeftEnabled=false; if Conns.autoLeft then Conns.autoLeft:Disconnect(); Conns.autoLeft=nil end; State.autoLeftPhase=1; if stackBtnRefs.autoLeft then stackBtnRefs.autoLeft.setOn(false) end; faceSouth(); return end
local d=(POS.L2-root.Position); local mv=Vector3.new(d.X,0,d.Z).Unit; hum2:Move(mv,false); root.AssemblyLinearVelocity=Vector3.new(mv.X*spd,root.AssemblyLinearVelocity.Y,mv.Z*spd)
end
end)
end
end
stopAutoLeft=function()
_apRouteRunning=false; _apRouteId=_apRouteId+1
if Conns.autoLeft then Conns.autoLeft:Disconnect(); Conns.autoLeft=nil end
State.autoLeftPhase=1
local c=LP.Character; if c then
local hum=c:FindFirstChildOfClass("Humanoid"); if hum then hum:Move(Vector3.zero,false) end
local r=c:FindFirstChild("HumanoidRootPart"); if r then r.AssemblyLinearVelocity=Vector3.zero end
end
if stackBtnRefs.autoLeft then stackBtnRefs.autoLeft.setOn(false) end
end

startAutoRight=function()
if Conns.autoRight then Conns.autoRight:Disconnect(); Conns.autoRight=nil end
if modeToggleState=="full" then
_apRouteRunning=false; _apRouteId=_apRouteId+1
task.spawn(function()
local plot=_apGetMyPlot()
local detectedSpawn=plot==3 and 1 or (plot==7 and 2 or nil)
local route=detectedSpawn==1 and _apRoutes.START_B or _apRoutes.START_A
_apWalkRoute(route,function()
State.autoRightEnabled=false
if stackBtnRefs.autoRight then stackBtnRefs.autoRight.setOn(false) end
-- Animal delivered — turn carry speed back off (full auto only)
if modeToggleState == "full" then
    State.speedToggled = false
    if stackBtnRefs.carrySpeed then stackBtnRefs.carrySpeed.setOn(false) end
end
end)
end)
else
State.autoRightPhase=1
Conns.autoRight=RunService.Heartbeat:Connect(function()
if not State.autoRightEnabled then return end
local c=LP.Character; if not c then return end
local root=c:FindFirstChild("HumanoidRootPart"); local hum2=c:FindFirstChildOfClass("Humanoid"); if not root or not hum2 then return end
local spd=State.laggerEnabled and State.laggerSpeed or State.normalSpeed
State._autoPlaySpeed = spd
hum2.WalkSpeed = spd
if State.autoRightPhase==1 then
local tgt=Vector3.new(POS.R1.X,root.Position.Y,POS.R1.Z); if (tgt-root.Position).Magnitude<1 then State.autoRightPhase=2; local d=(POS.R2-root.Position); local mv=Vector3.new(d.X,0,d.Z).Unit; hum2:Move(mv,false); root.AssemblyLinearVelocity=Vector3.new(mv.X*spd,root.AssemblyLinearVelocity.Y,mv.Z*spd); return end
local d=(POS.R1-root.Position); local mv=Vector3.new(d.X,0,d.Z).Unit; hum2:Move(mv,false); root.AssemblyLinearVelocity=Vector3.new(mv.X*spd,root.AssemblyLinearVelocity.Y,mv.Z*spd)
elseif State.autoRightPhase==2 then
local tgt=Vector3.new(POS.R2.X,root.Position.Y,POS.R2.Z); if (tgt-root.Position).Magnitude<1 then hum2:Move(Vector3.zero,false); root.AssemblyLinearVelocity=Vector3.zero; State.autoRightEnabled=false; if Conns.autoRight then Conns.autoRight:Disconnect(); Conns.autoRight=nil end; State.autoRightPhase=1; if stackBtnRefs.autoRight then stackBtnRefs.autoRight.setOn(false) end; faceNorth(); return end
local d=(POS.R2-root.Position); local mv=Vector3.new(d.X,0,d.Z).Unit; hum2:Move(mv,false); root.AssemblyLinearVelocity=Vector3.new(mv.X*spd,root.AssemblyLinearVelocity.Y,mv.Z*spd)
end
end)
end
end
stopAutoRight=function()
_apRouteRunning=false; _apRouteId=_apRouteId+1
if Conns.autoRight then Conns.autoRight:Disconnect(); Conns.autoRight=nil end
State.autoRightPhase=1
local c=LP.Character; if c then
local hum=c:FindFirstChildOfClass("Humanoid"); if hum then hum:Move(Vector3.zero,false) end
local r=c:FindFirstChild("HumanoidRootPart"); if r then r.AssemblyLinearVelocity=Vector3.zero end
end
if stackBtnRefs.autoRight then stackBtnRefs.autoRight.setOn(false) end
end

-- ============================================================
-- DUEL COUNTDOWN WATCHER
-- ============================================================
local Conns_duelWatch = nil
local function startDuelCountdownWatcher(direction)
if Conns_duelWatch then Conns_duelWatch:Disconnect(); Conns_duelWatch=nil end
State._duelWaiting = true

local function countdownVisible()
    local pg = LP:FindFirstChild("PlayerGui"); if not pg then return false end
    for _, obj in ipairs(pg:GetDescendants()) do
        if obj:IsA("TextLabel") then
            local t = obj.Text
            if t == "3" or t == "2" or t == "1" or t == "GO!" or t == "Go!" then
                if obj.Visible then return true end
            end
        end
        if obj:IsA("ScreenGui") then
            local n = obj.Name:lower()
            if (n:find("duel") or n:find("countdown") or n:find("battle")) and obj.Enabled then
                return true
            end
        end
    end
    return false
end

local sawCountdown = false
Conns_duelWatch = RunService.Heartbeat:Connect(function()
    if not State.duelCountdownEnabled then
        Conns_duelWatch:Disconnect(); Conns_duelWatch=nil; State._duelWaiting=false; return
    end
    local visible = countdownVisible()
    if not sawCountdown and visible then
        sawCountdown = true
    end
    if sawCountdown and not visible then
        Conns_duelWatch:Disconnect(); Conns_duelWatch=nil; State._duelWaiting=false
        task.wait(0.05)
        if direction == "left" then
            State.autoLeftEnabled=true
            if stackBtnRefs.autoLeft then stackBtnRefs.autoLeft.setOn(true) end
            startAutoLeft()
        elseif direction == "right" then
            State.autoRightEnabled=true
            if stackBtnRefs.autoRight then stackBtnRefs.autoRight.setOn(true) end
            startAutoRight()
        end
        task.delay(2, function()
            if State.duelCountdownEnabled then
                startDuelCountdownWatcher(direction)
            end
        end)
    end
end)
end

local function stopDuelCountdownWatcher()
if Conns_duelWatch then Conns_duelWatch:Disconnect(); Conns_duelWatch=nil end
State._duelWaiting = false
end

-- ============================================================
startAntiRagdoll=function()
if Conns.antiRag then return end
Conns.antiRag=RunService.Heartbeat:Connect(function()
if not State.antiRagdollEnabled then return end
local c=LP.Character; if not c then return end
local hum2=c:FindFirstChildOfClass("Humanoid"); local root=c:FindFirstChild("HumanoidRootPart")
if not hum2 or not root then return end; if hum2.Health<=0 then return end
local st=hum2:GetState(); if st==Enum.HumanoidStateType.Dead then return end
if st==Enum.HumanoidStateType.Physics or st==Enum.HumanoidStateType.Ragdoll or st==Enum.HumanoidStateType.FallingDown then
pcall(function() hum2:ChangeState(Enum.HumanoidStateType.GettingUp) end)
pcall(function() workspace.CurrentCamera.CameraSubject=hum2 end)
pcall(function() local PM=LP.PlayerScripts:FindFirstChild("PlayerModule"); if PM then local CM=require(PM:FindFirstChild("ControlModule")); if CM then CM:Enable() end end end)
root.Velocity=Vector3.new(0,0,0); root.RotVelocity=Vector3.new(0,0,0)
end
for _,obj in ipairs(c:GetDescendants()) do pcall(function() if obj:IsA("Motor6D") and obj.Enabled==false then obj.Enabled=true end end) end
end)
end
stopAntiRagdoll=function() if Conns.antiRag then Conns.antiRag:Disconnect(); Conns.antiRag=nil end end

-- ============================================================
-- FPS BOOST
-- ============================================================


-- ============================================================
-- STEAL
-- ============================================================
local function isMyPlotByName(pn)
local ct=tick(); if Steal.plotCache[pn] and (ct-(Steal.plotCacheTime[pn] or 0))<PLOT_CACHE_DURATION then return Steal.plotCache[pn] end
local plots=workspace:FindFirstChild("Plots"); if not plots then Steal.plotCache[pn]=false; Steal.plotCacheTime[pn]=ct; return false end
local plot=plots:FindFirstChild(pn); if not plot then Steal.plotCache[pn]=false; Steal.plotCacheTime[pn]=ct; return false end
local sign=plot:FindFirstChild("PlotSign"); if sign then local yb=sign:FindFirstChild("YourBase"); if yb and yb:IsA("BillboardGui") then local r=yb.Enabled==true; Steal.plotCache[pn]=r; Steal.plotCacheTime[pn]=ct; return r end end
Steal.plotCache[pn]=false; Steal.plotCacheTime[pn]=ct; return false
end
local function findNearestPrompt()
local c=LP.Character; if not c then return nil end; local root=c:FindFirstChild("HumanoidRootPart"); if not root then return nil end
local ct=tick(); if ct-Steal.promptCacheTime<PROMPT_CACHE_REFRESH and #Steal.cachedPrompts>0 then local np,nd=nil,math.huge; for _,data in ipairs(Steal.cachedPrompts) do if data.spawn then local dist=(data.spawn.Position-root.Position).Magnitude; if dist<=Steal.StealRadius and dist<nd then np=data.prompt; nd=dist end end end; if np then return np end end
Steal.cachedPrompts={}; Steal.promptCacheTime=ct; local plots=workspace:FindFirstChild("Plots"); if not plots then return nil end; local np,nd=nil,math.huge
for _,plot in ipairs(plots:GetChildren()) do if isMyPlotByName(plot.Name) then continue end; local pods=plot:FindFirstChild("AnimalPodiums"); if not pods then continue end
for _,pod in ipairs(pods:GetChildren()) do pcall(function() local base=pod:FindFirstChild("Base"); local sp=base and base:FindFirstChild("Spawn"); if sp then local att=sp:FindFirstChild("PromptAttachment"); if att then for _,child in ipairs(att:GetChildren()) do if child:IsA("ProximityPrompt") then local dist=(sp.Position-root.Position).Magnitude; table.insert(Steal.cachedPrompts,{prompt=child,spawn=sp}); if dist<=Steal.StealRadius and dist<nd then np=child; nd=dist end; break end end end end end) end
end; return np
end
local function executeSteal(prompt)
local ct=tick(); if ct-State.lastStealTick<STEAL_COOLDOWN then return end; if State.isStealing then return end
if not Steal.Data[prompt] then Steal.Data[prompt]={hold={},trigger={},ready=true}; pcall(function() if getconnections then for _,c2 in ipairs(getconnections(prompt.PromptButtonHoldBegan)) do if c2.Function then table.insert(Steal.Data[prompt].hold,c2.Function) end end; for _,c2 in ipairs(getconnections(prompt.Triggered)) do if c2.Function then table.insert(Steal.Data[prompt].trigger,c2.Function) end end else Steal.Data[prompt].useFallback=true end end) end
local data=Steal.Data[prompt]; if not data.ready then return end; data.ready=false; State.isStealing=true; State.stealStartTime=ct; State.lastStealTick=ct
if Conns.progress then Conns.progress:Disconnect() end
Conns.progress=RunService.Heartbeat:Connect(function()
if not State.isStealing then Conns.progress:Disconnect(); return end
local prog=math.clamp((tick()-State.stealStartTime)/Steal.StealDuration,0,1)
progressFill.Size=UDim2.new(prog,0,1,0)
stealPctLbl.Text=math.floor(prog*100).."%"
if prog<=0.33 then progressFill.BackgroundColor3=Color3.fromRGB(34,197,94)
elseif prog<=0.66 then progressFill.BackgroundColor3=Color3.fromRGB(0,255,128)
else progressFill.BackgroundColor3=Color3.fromRGB(57,255,180) end
end)
task.spawn(function()
local ok=false; pcall(function() if not data.useFallback then for _,fn in ipairs(data.hold) do task.spawn(fn) end; task.wait(Steal.StealDuration); for _,fn in ipairs(data.trigger) do task.spawn(fn) end; ok=true end end)
if not ok and fireproximityprompt then pcall(function() fireproximityprompt(prompt); ok=true end) end
if not ok then pcall(function() prompt:InputHoldBegin(); task.wait(Steal.StealDuration); prompt:InputHoldEnd() end) end
task.wait(Steal.StealDuration*0.3); if Conns.progress then Conns.progress:Disconnect() end; resetProgressBar(); task.wait(0.05); data.ready=true; State.isStealing=false

end)
end
-- ── Auto Steal ───────────────────────────────────────────────
startAutoSteal=function()
if Conns.autoSteal then return end
Conns.autoSteal=RunService.Heartbeat:Connect(function()
    if not Steal.AutoStealEnabled or State.isStealing then return end
    local ok, p = pcall(findNearestPrompt)
    if ok and p then pcall(executeSteal, p) end
end)
end
stopAutoSteal=function()
if Conns.autoSteal then Conns.autoSteal:Disconnect(); Conns.autoSteal=nil end
State.isStealing=false; State.lastStealTick=0; Steal.plotCache={}; Steal.plotCacheTime={}; Steal.cachedPrompts={}; resetProgressBar()
end

-- ============================================================
-- SAVE CONFIG
-- ============================================================
saveConfig = function()
local cfg = {
normalSpeed = State.normalSpeed,
carrySpeed = State.carrySpeed,
laggerSpeed = State.laggerSpeed,
stealRadius = Steal.StealRadius,
stealDuration = Steal.StealDuration,
uiScale = uiScaleObj and uiScaleObj.Scale or 1.0,
stackButtonsHidden = State.stackButtonsHidden,
uiLocked = State.uiLocked,
stealDurationVal = Steal.StealDuration,
speedKey = Keys.speed.Name,
autoLeftKey = Keys.autoLeft.Name,
autoRightKey = Keys.autoRight.Name,
guiHideKey = Keys.guiHide.Name,
dropKey = Keys.drop.Name,
laggerKey = Keys.lagger.Name,
tpDownKey = Keys.tpDown.Name,
aimbotKey = Keys.aimbot.Name,
infJump = State.infJumpEnabled,
antiRagdoll = State.antiRagdollEnabled,
medusaCounter = State.medusaCounterEnabled,
batCounter = State.batCounterEnabled,
autoStealEnabled = Steal.AutoStealEnabled,
noAnim = State.noAnimEnabled,
tryhardAnim = State.animEnabled,
stretchRez = State.stretchRezEnabled,
stretchRezFov = State.stretchRezFov,
nightTime = State.nightTimeEnabled,
purpleSky = State.purpleSkyEnabled,
fpsBooster = State.fpsBoosterEnabled,
espEnabled = State.espEnabled,
autoTpDown = State.autoTpDownEnabled,
autoTpDownHeight = State.autoTpDownHeight,
modeToggleState = modeToggleState,
autoSwingEnabled = State.autoSwingEnabled,
}
-- Save stack button positions
local btnPositions = {}
for key, wrapper in pairs(stackWrappers) do
    local pos = wrapper.Position
    btnPositions[key] = {sx=pos.X.Scale, ox=pos.X.Offset, sy=pos.Y.Scale, oy=pos.Y.Offset}
end
cfg.btnPositions = btnPositions
-- Save steal bar position
local ibp = infoBar.Position
cfg.infoBarPos = {sx=ibp.X.Scale, ox=ibp.X.Offset, sy=ibp.Y.Scale, oy=ibp.Y.Offset}
local ok, encoded = pcall(function() return HttpService:JSONEncode(cfg) end)
if ok then pcall(function() _writefile(CONFIG_FILE, encoded) end) end
end

-- ============================================================
-- LOAD CONFIG
-- ============================================================
loadConfig = function()
local hasFile = false
pcall(function() hasFile = _isfile(CONFIG_FILE) end)
if not hasFile then pcall(function() hasFile = _isfile("RadiantHubConfig.json") end) end
if not hasFile then return end

local raw
local ok = pcall(function() raw = _readfile(CONFIG_FILE) end)
if not ok or not raw then pcall(function() raw = _readfile("RadiantHubConfig.json") end) end
if not raw then return end

local cfg; local ok2 = pcall(function() cfg = HttpService:JSONDecode(raw) end)
if not ok2 or not cfg then return end

if cfg.normalSpeed then State.normalSpeed = cfg.normalSpeed; if normalBox then normalBox.Text = tostring(cfg.normalSpeed) end end
if cfg.carrySpeed  then State.carrySpeed  = cfg.carrySpeed;  if carryBox  then carryBox.Text  = tostring(cfg.carrySpeed)  end end
if cfg.laggerSpeed then State.laggerSpeed = cfg.laggerSpeed; if laggerBox then laggerBox.Text = tostring(cfg.laggerSpeed) end end

if cfg.stealRadius   then Steal.StealRadius   = cfg.stealRadius   end
if cfg.stealDuration    then Steal.StealDuration = cfg.stealDuration end
if cfg.stealDurationVal then Steal.StealDuration = cfg.stealDurationVal end
if stealDurBox then stealDurBox.Text = tostring(Steal.StealDuration) end
if cfg.uiLocked         then State.uiLocked = true end

if cfg.uiScale and uiScaleObj then
    uiScaleObj.Scale = cfg.uiScale
    if uiScaleBox then uiScaleBox.Text = tostring(cfg.uiScale) end
end
if cfg.stackButtonsHidden then
    applyStackButtonsVisible(false)
    if setHideButtonsToggle then setHideButtonsToggle(true) end
end

local function tryKey(field, keyTarget)
    if cfg[field] and Enum.KeyCode[cfg[field]] then
        local kc = Enum.KeyCode[cfg[field]]
        Keys[keyTarget] = kc
        if keybindBtnRefs[keyTarget] then
            keybindBtnRefs[keyTarget].Text = getKeyDisplayName(kc)
        end
    end
end
tryKey("speedKey",    "speed")
tryKey("autoLeftKey", "autoLeft")
tryKey("autoRightKey","autoRight")
tryKey("guiHideKey",  "guiHide")
tryKey("dropKey",     "drop")
tryKey("laggerKey",   "lagger")
tryKey("tpDownKey",   "tpDown")
tryKey("aimbotKey",   "aimbot")

-- Restore toggle states (values first, then update pill buttons)
if cfg.autoStealEnabled then Steal.AutoStealEnabled = true end
if cfg.infJump          then State.infJumpEnabled = true end
if cfg.antiRagdoll      then State.antiRagdollEnabled = true end
if cfg.medusaCounter    then State.medusaCounterEnabled = true end
if cfg.batCounter       then State.batCounterEnabled = true end
if cfg.noAnim           then State.noAnimEnabled = true end
if cfg.tryhardAnim      then State.animEnabled = true end
if cfg.stretchRez       then State.stretchRezEnabled = true end
if cfg.stretchRezFov    then
    local v = math.clamp(math.floor(cfg.stretchRezFov), 0, 120)
    State.stretchRezFov = v
    if stretchRezFovBox then stretchRezFovBox.Text = tostring(v) end
end
if cfg.nightTime        then State.nightTimeEnabled = true end
if cfg.purpleSky        then State.purpleSkyEnabled = true end
if cfg.fpsBooster       then State.fpsBoosterEnabled = true end
if cfg.espEnabled       then State.espEnabled = true end
if cfg.autoTpDown       then State.autoTpDownEnabled = true end
if cfg.autoTpDownHeight then State.autoTpDownHeight = cfg.autoTpDownHeight end
if cfg.autoSwingEnabled then State.autoSwingEnabled = true end
if cfg.modeToggleState  then modeToggleState = cfg.modeToggleState end

-- Update pill buttons to reflect loaded state
task.defer(function()
    if cfg.autoStealEnabled and setInstaGrab     then setInstaGrab(true);     pcall(startAutoSteal) end
    if cfg.infJump          and setInfJump       then setInfJump(true)       end
    if cfg.antiRagdoll      and setAntiRag       then setAntiRag(true);      startAntiRagdoll()    end
    if cfg.medusaCounter    and setMedusaCounter then setMedusaCounter(true) end
    if cfg.batCounter       and setBatCounter    then setBatCounter(true);   startBatCounter()     end
    if cfg.noAnim           and setNoAnim        then setNoAnim(true, true); task.spawn(function() local char = LP.Character or LP.CharacterAdded:Wait(); pcall(function() startNoAnim(char) end) end) end
    if cfg.tryhardAnim      and setTryhardAnim   then setTryhardAnim(true, true); task.spawn(function() task.wait(0.5); startAnimToggle() end) end
    if cfg.stretchRez and setStretchRez then
        setStretchRez(true, true)
        local cam = workspace.CurrentCamera
        if cam then cam.FieldOfView = State.stretchRezFov end
    end
    if cfg.nightTime and setNightTime then setNightTime(true) end
    if cfg.purpleSky and setPurpleSky then setPurpleSky(true) end
    if cfg.autoTpDown and setAutoTpDown then setAutoTpDown(true) end
    if cfg.autoTpDownHeight and autoTpDownHeightBox then autoTpDownHeightBox.Text = tostring(cfg.autoTpDownHeight) end
    if cfg.autoSwingEnabled and setAutoSwing then setAutoSwing(true) end
    if cfg.modeToggleState then
        if modeBtn then modeBtn.Text = modeToggleState == "semi" and "Mode: Semi Auto Play" or "Mode: Full Auto Play" end
    end
    if cfg.fpsBooster and setFpsBooster then
        setFpsBooster(true)
    end
    -- ESP: let setV call the callback so State is set AND highlights applied
    if cfg.espEnabled and setPlayerESP then setPlayerESP(true) end
    if cfg.uiLocked then State.uiLocked = true; if setLockUI then setLockUI(true, true) end end
    -- Restore steal bar position
    if cfg.infoBarPos then
        local p = cfg.infoBarPos
        infoBar.Position = UDim2.new(p.sx or 0.5, p.ox or -130, p.sy or 0, p.oy or 35)
    end
    if cfg.btnPositions then
        task.wait(0.1)
        -- Check if positions look valid (spread out, not all same spot)
        local ox_vals = {}
        local oy_vals = {}
        for key, pos in pairs(cfg.btnPositions) do
            table.insert(ox_vals, pos.ox or 0)
            table.insert(oy_vals, pos.oy or 0)
        end
        -- Find spread of x positions
        local minX, maxX = math.huge, -math.huge
        for _, v in ipairs(ox_vals) do minX = math.min(minX, v); maxX = math.max(maxX, v) end
        local spreadX = maxX - minX
        -- Only restore if buttons are spread out (not all stacked)
        if spreadX >= BTN_W then
            for key, pos in pairs(cfg.btnPositions) do
                local wrapper = stackWrappers[key]
                if wrapper then
                    wrapper.Position = UDim2.new(pos.sx or 0, pos.ox or 0, pos.sy or 0, pos.oy or 0)
                end
            end
        end
        -- else: leave default grid positions alone
    end
    -- Medusa counter needs character — wait for it
    if cfg.medusaCounter then
        task.spawn(function()
            local char = LP.Character or LP.CharacterAdded:Wait()
            setupMedusaCounter(char)
        end)
    end
    -- Fix billboard label on rejoin: loadConfig runs after setupChar so update it now
    local char = LP.Character
    if char then
        local head = char:FindFirstChild("Head")
        local bb = head and head:FindFirstChild("RadiantHubBB")
        local modeLbl = bb and bb:FindFirstChild("ModeBillLbl")
        if modeLbl then
            modeLbl.Text = modeToggleState == "semi" and "Mode: Semi" or "Mode: Full"
        end
    end
end)

-- Refresh all input boxes to show loaded values
task.defer(function()
    if normalBox   then normalBox.Text   = tostring(State.normalSpeed)    end
    if carryBox    then carryBox.Text    = tostring(State.carrySpeed)     end
    if laggerBox   then laggerBox.Text   = tostring(State.laggerSpeed)    end
    if stealRadBox then stealRadBox.Text = tostring(Steal.StealRadius)    end
    if radTB and not radTB:IsFocused() then radTB.Text = tostring(Steal.StealRadius) end
    if uiScaleBox  then uiScaleBox.Text  = tostring(uiScaleObj and uiScaleObj.Scale or 1.0) end
    -- Restore steal duration box if it exists
    for _, page in ipairs(contentFrame and contentFrame:GetChildren() or {}) do
        for _, child in ipairs(page:GetChildren()) do
            if child:IsA("Frame") then
                local lbl = child:FindFirstChildWhichIsA("TextLabel")
                local box = child:FindFirstChildWhichIsA("TextBox")
                if lbl and box and lbl.Text == "Steal Duration" then
                    box.Text = tostring(Steal.StealDuration)
                end
            end
        end
    end
end)
end

-- ============================================================
-- NO ANIMATION FUNCTIONS
-- ============================================================
local _noAnimConn = nil
local _noAnimAnimConn = nil
local _savedAnimScript = nil  -- backup of Animate script before we destroy it

startNoAnim = function(character, humanoid)
    character = character or LP.Character
    humanoid = humanoid or (character and character:FindFirstChildOfClass("Humanoid"))
    if not character or not humanoid then return end

    -- Disconnect any existing connections first
    if _noAnimConn then _noAnimConn:Disconnect(); _noAnimConn = nil end
    if _noAnimAnimConn then _noAnimAnimConn:Disconnect(); _noAnimAnimConn = nil end

    -- Step 1: get or wait for Animator (needed to stop tracks)
    local animator = humanoid:FindFirstChildOfClass("Animator")
    if not animator then
        -- Wait up to 3s synchronously via a loop instead of WaitForChild
        -- so we don't block the entire thread
        task.spawn(function()
            local t = tick()
            repeat RunService.Heartbeat:Wait()
                animator = humanoid:FindFirstChildOfClass("Animator")
            until animator or tick() - t > 3
            if animator then
                for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
                    track:Stop(0)
                end
                if _noAnimAnimConn then _noAnimAnimConn:Disconnect() end
                _noAnimAnimConn = animator.AnimationPlayed:Connect(function(track)
                    track:Stop(0)
                end)
            end
        end)
    else
        -- Stop all tracks immediately right now
        for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
            track:Stop(0)
        end
        -- Block all future tracks
        if _noAnimAnimConn then _noAnimAnimConn:Disconnect() end
        _noAnimAnimConn = animator.AnimationPlayed:Connect(function(track)
            track:Stop(0)
        end)
    end

    -- Step 2: save and destroy Animate script immediately
    local animScript = character:FindFirstChild("Animate")
    if animScript then
        _savedAnimScript = animScript:Clone()
        _savedAnimScript.Disabled = true
        animScript:Destroy()
    end
    for _, v in ipairs(character:GetChildren()) do
        if v:IsA("LocalScript") and v.Name:lower() == "animate" then v:Destroy() end
    end

    -- Step 3: snapshot joint CFrames immediately (T-pose / current pose)
    local joints = {}
    for _, desc in ipairs(character:GetDescendants()) do
        if desc:IsA("Motor6D") and desc.Part1 then
            joints[#joints+1] = {motor=desc, c0=desc.C0, c1=desc.C1}
        end
    end

    -- Step 4: lock joints every Heartbeat — this runs immediately, no yield
    if _noAnimConn then _noAnimConn:Disconnect() end
    _noAnimConn = RunService.Heartbeat:Connect(function()
        if not character or not character.Parent then
            _noAnimConn:Disconnect(); _noAnimConn = nil
            return
        end
        for _, j in ipairs(joints) do
            if j.motor and j.motor.Parent then
                j.motor.C0 = j.c0
                j.motor.C1 = j.c1
            end
        end
    end)
end

stopNoAnim = function()
    -- Disconnect heartbeat joint lock and animation blocker
    if _noAnimConn then _noAnimConn:Disconnect(); _noAnimConn = nil end
    if _noAnimAnimConn then _noAnimAnimConn:Disconnect(); _noAnimAnimConn = nil end

    local char = LP.Character
    if not char then return end

    task.spawn(function()
        -- Restore using our saved clone of the original Animate script
        if _savedAnimScript then
            local newAnim = _savedAnimScript:Clone()
            newAnim.Disabled = false
            newAnim.Parent = char
            _savedAnimScript = nil
        else
            -- Fallback: try StarterCharacterScripts
            local starterChar = game:GetService("StarterPlayer"):FindFirstChild("StarterCharacterScripts")
            local animTemplate = starterChar and starterChar:FindFirstChild("Animate")
            -- Fallback: copy from another player
            if not animTemplate then
                for _, p in ipairs(game:GetService("Players"):GetPlayers()) do
                    if p ~= LP and p.Character then
                        local a = p.Character:FindFirstChild("Animate")
                        if a then animTemplate = a; break end
                    end
                end
            end
            if animTemplate then
                local newAnim = animTemplate:Clone()
                newAnim.Disabled = false
                newAnim.Parent = char
            else
                LP:LoadCharacter()
            end
        end
    end)
end

-- ============================================================
-- CHARACTER SETUP
-- ============================================================
setupChar = function(char)
h=char:WaitForChild("Humanoid",5)
hrp=char:WaitForChild("HumanoidRootPart",5)
if not h or not hrp then return end

-- Re-apply no-anim immediately on new character (no delay so death anim never starts)
if State.noAnimEnabled then pcall(startNoAnim, char, h) end
if State.animEnabled then task.spawn(function() task.wait(0.3); saveOriginalAnims(char); applyAnimPack(char) end) end

local head=char:FindFirstChild("Head")
if head then
    local oldBB=head:FindFirstChild("RadiantHubBB"); if oldBB then oldBB:Destroy() end

    local bb=Instance.new("BillboardGui", head)
    bb.Name="RadiantHubBB"
    bb.Size=UDim2.new(0, 180, 0, 60)
    bb.StudsOffset=Vector3.new(0, 3, 0)
    bb.AlwaysOnTop=true

    -- Speed label
    local speedBillLbl=Instance.new("TextLabel", bb)
    speedBillLbl.Name="SpeedBillLbl"
    speedBillLbl.Size=UDim2.new(1, 0, 0, 32)
    speedBillLbl.Position=UDim2.new(0, 0, 0, 10)
    speedBillLbl.BackgroundTransparency=1
    speedBillLbl.Text="0.0"
    speedBillLbl.TextColor3=Color3.fromRGB(255, 255, 255)
    speedBillLbl.Font=Enum.Font.GothamBlack
    speedBillLbl.TextScaled=true
    speedBillLbl.TextStrokeTransparency=0
    speedBillLbl.TextStrokeColor3=Color3.fromRGB(0, 0, 0)

    local modeLbl=Instance.new("TextLabel",bb)
    modeLbl.Name="ModeBillLbl"
    modeLbl.Size=UDim2.new(1,0,0,20)
    modeLbl.Position=UDim2.new(0,0,0,44)
    modeLbl.BackgroundTransparency=1
    if modeToggleState=="semi" then modeLbl.Text="Mode: Semi" else modeLbl.Text="Mode: Full" end
    modeLbl.TextColor3=Color3.fromRGB(255,255,255)
    modeLbl.Font=Enum.Font.GothamBlack
    modeLbl.TextScaled=true
    modeLbl.TextStrokeTransparency=0
    modeLbl.TextStrokeColor3=Color3.fromRGB(0,0,0)

end

stopAntiRagdoll()
if State.antiRagdollEnabled then task.wait(0.5); startAntiRagdoll() end
if State.medusaCounterEnabled then setupMedusaCounter(char) end
if State.batCounterEnabled then task.wait(0.3); startBatCounter() end

-- Re-apply FPS booster: strip accessories from new character and all others
if State.fpsBoosterEnabled then
    task.wait(0.2)
    local Lighting = game:GetService("Lighting")
    settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
    Lighting.GlobalShadows = false
    Lighting.Brightness = 3
    Lighting.FogEnd = 9000000000
    Lighting.Technology = Enum.Technology.Legacy
    Lighting.EnvironmentDiffuseScale = 0
    Lighting.EnvironmentSpecularScale = 0
    for _, plr in ipairs(Players:GetPlayers()) do
        local c = plr.Character
        if c then
            for _, acc in ipairs(c:GetChildren()) do
                if acc:IsA("Accessory") then acc:Destroy() end
            end
        end
    end
end

end

LP.CharacterAdded:Connect(setupChar)
if LP.Character then task.spawn(function() setupChar(LP.Character) end) end

-- ============================================================
-- RUNTIME LOOPS
-- ============================================================
-- Throttled collision disabler: runs every 0.5s instead of every physics step
task.spawn(function()
while true do
task.wait(0.5)
for _,p in ipairs(Players:GetPlayers()) do
if p~=LP and p.Character then
for _,part in ipairs(p.Character:GetChildren()) do
if part:IsA("BasePart") and part.CanCollide then part.CanCollide=false end
end
end
end
end
end)

UIS.JumpRequest:Connect(function()
if not State.infJumpEnabled then return end
local c=LP.Character; if not c then return end; local root=c:FindFirstChild("HumanoidRootPart")
if root then root.Velocity=Vector3.new(root.Velocity.X,55,root.Velocity.Z) end
end)

-- Speed billboard label cache to avoid lookups every frame
local _lastBillUpdate = 0
local _lastHeartbeatDt = 1/60
local _displayedSpeed = 0  -- smoothly interpolated speed value

RunService.Heartbeat:Connect(function(dt)
_lastHeartbeatDt = dt

-- Billboard speed label: always runs regardless of h/hrp so it recovers after reset
do
    local char = LP.Character
    if char ~= _billLastChar then
        _billLastChar = char
        _billLbl = nil
    end
    if char and not _billLbl then
        local head2 = char:FindFirstChild("Head")
        local bb2 = head2 and head2:FindFirstChild("RadiantHubBB")
        _billLbl = bb2 and bb2:FindFirstChild("SpeedBillLbl")
    end
    if _billLbl then
        _billLbl.Text = string.format("%.1f", _displayedSpeed)
    end
end

if not (h and hrp) then return end; if State._tpInProgress then return end

if not State.autoLeftEnabled and not State.autoRightEnabled then
    local md=h.MoveDirection
    local spd
    if State.laggerEnabled then
        spd = State.laggerSpeed
    elseif State.speedToggled then
        spd = State.carrySpeed
    else
        spd = State.normalSpeed
    end
    if md.Magnitude>0 then State.lastMoveDir=md; hrp.Velocity=Vector3.new(md.X*spd,hrp.Velocity.Y,md.Z*spd)
    elseif State.antiRagdollEnabled and State.lastMoveDir.Magnitude>0 then
        local anyHeld=false; for key in pairs(MOVE_KEYS) do if UIS:IsKeyDown(key) then anyHeld=true; break end end
        if anyHeld then hrp.Velocity=Vector3.new(State.lastMoveDir.X*spd,hrp.Velocity.Y,State.lastMoveDir.Z*spd) end
    end
elseif (State.autoLeftEnabled or State.autoRightEnabled) then
    local spd = State._autoPlaySpeed
    local curVel = hrp.Velocity
    local flatVel = Vector3.new(curVel.X, 0, curVel.Z)
    if flatVel.Magnitude > 0.5 then
        local dir = flatVel.Unit
        hrp.Velocity = Vector3.new(dir.X*spd, curVel.Y, dir.Z*spd)
    end
end

-- Instant speed display
local realSpd = Vector3.new(hrp.Velocity.X, 0, hrp.Velocity.Z).Magnitude
_displayedSpeed = realSpd < 0.5 and 0 or realSpd

-- Auto TP Down: teleport to ground when player Y >= autoTpDownHeight
-- HRP sits ~3 studs above feet, so subtract offset so the input value matches actual height felt
if State.autoTpDownEnabled and hrp then
    local curY = hrp.Position.Y - 3
    if curY >= State.autoTpDownHeight then
        local rot = hrp.CFrame.Rotation
        hrp.CFrame = CFrame.new(hrp.Position.X, -8.80, hrp.Position.Z) * rot
    end
end
end)

-- ============================================================
-- INPUT
-- ============================================================
UIS.InputBegan:Connect(function(inp,gp)
if gp then return end
local isKb=inp.UserInputType==Enum.UserInputType.Keyboard
local isGp=inp.UserInputType==Enum.UserInputType.Gamepad1 or inp.UserInputType==Enum.UserInputType.Gamepad2 or inp.UserInputType==Enum.UserInputType.Gamepad3 or inp.UserInputType==Enum.UserInputType.Gamepad4
if not isKb and not isGp then return end
local kc=inp.KeyCode; if kc==Enum.KeyCode.Unknown then return end

if kc==Keys.speed then
    State.speedToggled=not State.speedToggled
    if stackBtnRefs.carrySpeed then stackBtnRefs.carrySpeed.setOn(State.speedToggled) end
elseif kc==Keys.autoLeft then
    State.autoLeftEnabled=not State.autoLeftEnabled
    if stackBtnRefs.autoLeft then stackBtnRefs.autoLeft.setOn(State.autoLeftEnabled) end
    if State.autoLeftEnabled then startAutoLeft() else stopAutoLeft() end
elseif kc==Keys.autoRight then
    State.autoRightEnabled=not State.autoRightEnabled
    if stackBtnRefs.autoRight then stackBtnRefs.autoRight.setOn(State.autoRightEnabled) end
    if State.autoRightEnabled then startAutoRight() else stopAutoRight() end
elseif kc==Keys.drop then
    if not State.dropEnabled then runDropBrainrot() end
elseif kc==Keys.lagger then
    State.laggerEnabled = not State.laggerEnabled
    if stackBtnRefs.lagger then stackBtnRefs.lagger.setOn(State.laggerEnabled) end
    if State.laggerEnabled then
        State._prevCarry = State.carrySpeed
        State._prevSpeed = State.speedToggled
        State.speedToggled = false
        if stackBtnRefs.carrySpeed then stackBtnRefs.carrySpeed.setOn(false) end
        if carryBox then carryBox.Text = tostring(State.laggerSpeed) end
    else
        State.carrySpeed = State._prevCarry or 30
        State.speedToggled = State._prevSpeed or false
        if carryBox then carryBox.Text = tostring(State.carrySpeed) end
        if stackBtnRefs.carrySpeed then stackBtnRefs.carrySpeed.setOn(State.speedToggled) end
    end
elseif kc==Keys.tpDown then
    doTpDown()
elseif kc==Keys.aimbot then
    State.batAimbotToggled = not State.batAimbotToggled
    if stackBtnRefs.aimbot then stackBtnRefs.aimbot.setOn(State.batAimbotToggled) end
    if State.batAimbotToggled then enableAutoBat() else disableAutoBat() end
elseif kc==Keys.guiHide then
    if isKb then State.guiVisible=not State.guiVisible; mainOuter.Visible=State.guiVisible end
end
end)

-- ============================================================
-- INIT
-- ============================================================
-- Load config AFTER UI is fully built so all pill setters and boxes exist
task.defer(function()
    loadConfig()
end)

-- Auto-save config every 5 seconds to persist any changes
task.spawn(function()
    while true do
        task.wait(5)
        pcall(saveConfig)
    end
end)

print("[Radiant Hub v6] Loaded")
